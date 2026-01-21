# Adding a New Stakeholder Type with Contact Information

This guide explains how to add a new stakeholder type to the Consultation Tools when the stakeholder has associated contact information (name, address, postal code, etc.).

---

## Built-in "Other" Stakeholder Types

The tool includes three built-in stakeholder types for custom/local data with embedded contacts:

| Name | Geometry | Expected Fields |
|------|----------|-----------------|
| `Other - Polygons` | Polygon | `Stakeholder_ID`, `Contact_Name`, `Address`, `City`, `Province`, `Postal_Code`, `Email`, `Phone` |
| `Other - Lines` | Line | `Stakeholder_ID`, `Contact_Name`, `Address`, `City`, `Province`, `Postal_Code`, `Email`, `Phone` |
| `Other - Points` | Point | `Stakeholder_ID`, `Contact_Name`, `Address`, `City`, `Province`, `Postal_Code`, `Email`, `Phone` |

**To use these**, simply add them to `util/stakeholders.csv` with the path to your local geodatabase feature class:

```csv
NAME,SOURCE,PATH,SQL,BUFFER,OVERLAY_FIELD
Other - Polygons,OTHER,\\path\to\your.gdb\Stakeholder_Polygon,,0,Stakeholder_ID
Other - Points,OTHER,\\path\to\your.gdb\Stakeholder_Point,,0,Stakeholder_ID
Other - Lines,OTHER,\\path\to\your.gdb\Stakeholder_Line,,0,Stakeholder_ID
```

**Important**: Your feature class must have fields with these exact names (case-sensitive):
- `Stakeholder_ID` - the unique identifier for each stakeholder
- `Contact_Name`, `Address`, `City`, `Province`, `Postal_Code`, `Email`, `Phone` - contact info fields

If your field names differ, you can override them in `config.json`:

```json
{
  "stakeholder_fields": {
    "other_polygons": ["YOUR_ID", "NAME", "ADDR", "CITY", "PROV", "POSTAL", "EMAIL", "PHONE"]
  }
}
```

---

## Overview

The tool retrieves stakeholder contact information from three possible sources:

| Source | When Used | Examples |
|--------|-----------|----------|
| **Embedded in feature class** | Contact fields exist in the BCGW/source data | POD (water licences), POD Applications |
| **Separate contact file** | You have a contact list matched by stakeholder ID | Guides, Trappers, Woodlots |
| **Common addresses file** | One contact applies to all IDs of that type | Regional District, Recreation District |

Choose the method that matches your data situation.

---

## Method 1: Contact Data Embedded in Feature Class

Use this method when the BCGW (or other) feature class already contains contact fields like name, address, and postal code.

### Step 1: Add to stakeholders.csv

```csv
NAME,SOURCE,PATH,SQL,BUFFER,OVERLAY_FIELD
Your New Type,BCGW,WHSE_SCHEMA.YOUR_FEATURE_CLASS_SVW,STATUS = 'ACTIVE',500,YOUR_ID_FIELD
```

### Step 2: Configure the Field List

Add your stakeholder's contact fields to the configuration. You have two options:

#### Option A: Add to `config/defaults.py` (Recommended)

Find the `STAKEHOLDER_FIELDS` dictionary and add your field list:

```python
STAKEHOLDER_FIELDS = {
    # ... existing entries ...
    'your_type': ['YOUR_ID_FIELD', 'CONTACT_NAME', 'ADDRESS',
                  'CITY', 'PROVINCE', 'POSTAL_CODE'],
}
```

#### Option B: Add to `config.json` (For local overrides)

```json
{
  "stakeholder_fields": {
    "your_type": ["YOUR_ID_FIELD", "CONTACT_NAME", "ADDRESS",
                  "CITY", "PROVINCE", "POSTAL_CODE"]
  }
}
```

### Step 3: Modify the Code (Required)

You must add special handling in two files to pull the contact fields:

#### 3a. Edit `analysis/stakeholders.py`

Find the `_init_stakeholder_fields` method (around line 47) and add your field list loading:

```python
def _init_stakeholder_fields(self):
    """Initialize stakeholder-specific field lists."""
    if self.config is not None:
        # ... existing code ...
        self.your_type_fields = self.config.get_stakeholder_fields('your_type')
    else:
        # ... existing code ...
        self.your_type_fields = ['YOUR_ID_FIELD', 'CONTACT_NAME', 'ADDRESS',
                                  'CITY', 'PROVINCE', 'POSTAL_CODE']
```

Find the `find_overlapping_stakeholders` method (around line 230) and add your type:

```python
# Add stakeholder-specific fields
if sh_name == 'POD Applications':
    fields += self.pod_app_fields
elif sh_name == 'POD':
    fields += self.pod_lic_fields
# ... existing elif blocks ...
elif sh_name == 'Your New Type':  # ADD THIS
    fields += self.your_type_fields
```

#### 3b. Edit `contacts/contact_builder.py`

Find the `get_contacts` method (around line 354) and add handling for your type:

```python
elif sh_name == 'Your New Type':
    contact_name = accessor.get_str('CONTACT_NAME')
    address = accessor.get_str('ADDRESS', '').strip()
    city = accessor.get_str('CITY', '').strip()
    province = accessor.get_str('PROVINCE', '').strip()
    postal_code = accessor.get_str('POSTAL_CODE', '').strip()
```

### When to Use This Method

- The source feature class already has contact columns
- Contact info is maintained in the authoritative data source
- You don't want to maintain a separate contact list

---

## Method 2: Separate Contact File (Recommended)

Use this method when you have a contact list that maps stakeholder IDs to contact information. This is the most flexible approach.

### Step 1: Add to stakeholders.csv

```csv
NAME,SOURCE,PATH,SQL,BUFFER,OVERLAY_FIELD
Your New Type,BCGW,WHSE_SCHEMA.YOUR_FEATURE_CLASS_SVW,STATUS = 'ACTIVE',500,YOUR_ID_FIELD
```

### Step 2: Create Contact File

Create an Excel file in `util/sh_contacts/` with a descriptive name:

**File**: `util/sh_contacts/your_type_contacts.xlsx`

| ID | NAME | ADDRESS | CITY | PROVINCE | POSTAL_CODE | EMAIL |
|----|------|---------|------|----------|-------------|-------|
| ABC123 | Jane Smith | 123 Main St | Kelowna | BC | V1Y 1A1 | jane@example.com |
| DEF456 | John Doe | 456 Oak Ave | Vernon | BC | V1T 2B2 | john@example.com |

#### Column Requirements

| Column | Required | Description |
|--------|----------|-------------|
| `ID` | Yes | Must match the `OVERLAY_FIELD` values from the spatial data |
| `NAME` | Yes | Contact person or organization name |
| `ADDRESS` | No | Street address |
| `CITY` | No | City name |
| `PROVINCE` | No | Province abbreviation (e.g., BC, AB) |
| `POSTAL_CODE` | No | Canadian postal code (e.g., V1Y 1A1) |
| `EMAIL` | No | Email address |

### Step 3: Modify contact_builder.py

Add a dictionary for your contact type and loading function.

#### 3a. Add dictionary in `__init__` method (around line 117):

```python
self.dict_your_type: Dict[str, Contact] = {}
```

#### 3b. Add loading function (add after `_load_trappers` method):

```python
def _load_your_type(self, filepath: str):
    """Load your type contacts from file."""
    try:
        df = pandas.read_excel(filepath, sheet_name=0)
        df = df.replace({numpy.nan: None})

        for i in df.index:
            id_val = safe_str(df['ID'][i]) if 'ID' in df.columns else ''
            if id_val:
                self.dict_your_type[id_val] = Contact(
                    name=df['NAME'][i] if 'NAME' in df.columns else '',
                    email=df['EMAIL'][i] if 'EMAIL' in df.columns else '',
                    address=df['ADDRESS'][i] if 'ADDRESS' in df.columns else '',
                    city=df['CITY'][i] if 'CITY' in df.columns else '',
                    province=df['PROVINCE'][i] if 'PROVINCE' in df.columns else '',
                    postal_code=df['POSTAL_CODE'][i] if 'POSTAL_CODE' in df.columns else ''
                )
    except Exception as e:
        self._log('Error loading your type contacts: {}'.format(e), 'warning')
```

#### 3c. Add file detection in `build_sh_contacts` (around line 259):

```python
elif re.search(r'your.?type', f, re.IGNORECASE):
    self._load_your_type(filepath)
```

#### 3d. Add contact lookup in `get_contacts` (around line 432):

```python
elif sh_name == 'Your New Type':
    if sh_id in self.dict_your_type:
        contact = self.dict_your_type[sh_id]
        contact_name = contact.name
        address = contact.address
        postal_code = contact.postal_code
        province = contact.province
        city = contact.city or self.postal_codes.get_city(postal_code)
        email = contact.email
```

### When to Use This Method

- You have a list of contacts matched to stakeholder IDs
- Contacts need to be updated independently of the spatial data
- Different business areas may have different contact lists

---

## Method 3: Common Addresses File

Use this method when all stakeholders of a type share the same contact (e.g., all Recreation Districts go to the same ministry office).

### Step 1: Add to stakeholders.csv

```csv
NAME,SOURCE,PATH,SQL,BUFFER,OVERLAY_FIELD
Your New Type,BCGW,WHSE_SCHEMA.YOUR_FEATURE_CLASS_SVW,,100,YOUR_ID_FIELD
```

### Step 2: Add to Common Addresses File

Create or edit `util/sh_contacts/common_addresses.xlsx`:

| TYPE | ID | NAME | ADDRESS | CITY | PROVINCE | POSTAL_CODE | EMAIL |
|------|-----|------|---------|------|----------|-------------|-------|
| Your New Type | All | Ministry Contact | 441 Columbia St | Kamloops | BC | V2C 2T3 | ministry@gov.bc.ca |

- Set `ID` to `All` if one contact applies to all stakeholders of that type
- Set `ID` to a specific value if different IDs have different contacts

### When to Use This Method

- All stakeholders of a type go to the same contact
- Contact info is generic (e.g., a ministry office)
- You don't need individual contact tracking

---

## Complete Example: Adding "Mining Claims"

Let's walk through adding mining claims as a new stakeholder type with contacts from a separate file.

### Step 1: Add to stakeholders.csv

```csv
Mining Claims,BCGW,WHSE_MINERAL_TENURE.MTA_MINERAL_CLAIM_POLYGON_SVW,CLAIM_STATUS = 'GOOD',500,TENURE_NUMBER
```

### Step 2: Create Contact File

Create `util/sh_contacts/mining_contacts.xlsx`:

| ID | NAME | ADDRESS | CITY | PROVINCE | POSTAL_CODE | EMAIL |
|----|------|---------|------|----------|-------------|-------|
| 123456 | ABC Mining Ltd | 100 Mining Way | Vancouver | BC | V6B 1A1 | info@abcmining.com |
| 234567 | XYZ Resources | 200 Resource Rd | Kamloops | BC | V2C 2T3 | contact@xyz.com |

### Step 3: Edit contacts/contact_builder.py

Add to `__init__`:
```python
self.dict_mining: Dict[str, Contact] = {}
```

Add loading method:
```python
def _load_mining(self, filepath: str):
    """Load mining contacts from file."""
    try:
        df = pandas.read_excel(filepath, sheet_name=0)
        df = df.replace({numpy.nan: None})
        for i in df.index:
            id_val = safe_str(df['ID'][i]) if 'ID' in df.columns else ''
            if id_val:
                self.dict_mining[id_val] = Contact(
                    name=df['NAME'][i] if 'NAME' in df.columns else '',
                    email=df['EMAIL'][i] if 'EMAIL' in df.columns else '',
                    address=df['ADDRESS'][i] if 'ADDRESS' in df.columns else '',
                    city=df['CITY'][i] if 'CITY' in df.columns else '',
                    province=df['PROVINCE'][i] if 'PROVINCE' in df.columns else '',
                    postal_code=df['POSTAL_CODE'][i] if 'POSTAL_CODE' in df.columns else ''
                )
    except Exception as e:
        self._log('Error loading mining contacts: {}'.format(e), 'warning')
```

Add to `build_sh_contacts` file detection:
```python
elif re.search(r'mining', f, re.IGNORECASE):
    self._load_mining(filepath)
```

Add to `get_contacts`:
```python
elif sh_name == 'Mining Claims':
    if sh_id in self.dict_mining:
        contact = self.dict_mining[sh_id]
        contact_name = contact.name
        address = contact.address
        postal_code = contact.postal_code
        province = contact.province
        city = contact.city or self.postal_codes.get_city(postal_code)
        email = contact.email
```

### Step 4: Test

1. Run Consultation Package with STAKEHOLDERS analysis
2. Check the output Excel for your new stakeholder type
3. Verify contact information appears in the Contacts sheet
4. Run Consultation Letters to verify letters generate correctly

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| No contacts appearing | ID mismatch | Ensure `ID` column in contact file matches `OVERLAY_FIELD` values exactly |
| Contact file not loading | Wrong filename pattern | Check the regex pattern in `build_sh_contacts` matches your filename |
| Missing columns in output | Fields not added | Verify you added the fields to `find_overlapping_stakeholders` |
| KeyError for field | Field doesn't exist | Check field name spelling matches the feature class exactly |

---

## File Summary

| File | Purpose |
|------|---------|
| `util/stakeholders.csv` | Defines spatial data source and buffer distance |
| `util/sh_contacts/*.xlsx` | Contact files matched by stakeholder ID |
| `contacts/contact_builder.py` | Loads contacts and matches to stakeholders |
| `analysis/stakeholders.py` | Pulls additional fields from feature classes |
