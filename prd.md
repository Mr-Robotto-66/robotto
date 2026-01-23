# Product Requirements Document: ArcPro KML Export Toolbox

## 1. Overview

### 1.1 Product Name
**KML Export Tools**
- Toolbox: `KMLExportTools.pyt`
- Tool: "Layer to KML"

### 1.2 Purpose
A Python Toolbox (.pyt) for ArcGIS Pro that exports feature layers and annotation to KML/KMZ format with full styling preservation, exceeding the capabilities of XTools Pro's Layer to KML tool.

### 1.3 Target Platform
- ArcGIS Pro 2.9+ (Python 3.x with arcpy)
- Windows 10/11

---

## 2. Functional Requirements

### 2.1 Supported Layer Types
| Layer Type | Support Level |
|------------|---------------|
| Point | Full |
| Polyline | Full |
| Polygon | Full |
| Multipoint | Full |
| Annotation | Full |

### 2.2 Output Formats
- **KML** (.kml) - Uncompressed XML format
- **KMZ** (.kmz) - Compressed format with embedded resources (icons, images)
- User selects output format via tool parameter

### 2.3 Coordinate System Handling
- Automatic reprojection to WGS84 (EPSG:4326) - required by KML specification
- Support input layers in any projected or geographic coordinate system
- Preserve Z values when present

### 2.4 Symbology Conversion
Convert ArcGIS Pro symbology to KML styles:

| ArcGIS Symbology Type | KML Conversion |
|-----------------------|----------------|
| Simple Symbol | Single KML Style |
| Unique Values | Multiple KML Styles by category |
| Graduated Colors | KML Styles with color ramp |
| Graduated Symbols | KML Styles with size variation |
| Proportional Symbols | KML Styles with icon scaling |

**Style Elements to Convert:**
- Point symbols → KML IconStyle (embed custom icons in KMZ)
- Line symbols → KML LineStyle (color, width)
- Polygon symbols → KML PolyStyle (fill color, outline)
- Transparency/opacity values
- Symbol rotation

### 2.5 Label Export
- Export feature labels as KML LabelStyle
- Preserve label text from label expression
- Convert label symbology (font, size, color) to KML

### 2.6 Attribute Handling
- **HTML Popup Generation**: Format attributes as HTML tables in KML `<description>`
- **Field Selection**: User can select which fields to include
- **Field Alias Support**: Use field aliases in popup headers
- **Null Value Handling**: Option to hide or show null/empty values

### 2.7 Folder Organization
- Group features into KML Folders based on attribute field values
- Hierarchical folder structure support
- Custom folder naming

### 2.8 3D/Altitude Support
| Altitude Mode | Description |
|---------------|-------------|
| clampToGround | Default - features follow terrain |
| relativeToGround | Height above terrain surface |
| absolute | Height above sea level (MSL) |

- Use Z values from geometry when available
- Option to specify altitude field from attributes
- Extrude option for 3D visualization

### 2.9 Time Animation Support
- TimeSpan: Features visible during date range
- TimeStamp: Features appear at specific moment
- Support for date/datetime fields
- Time slider compatibility in Google Earth

### 2.10 Batch Processing
- Process multiple layers in single operation
- Options:
  - Combine all layers into single KML/KMZ
  - Export each layer to separate file
- Progress reporting for large datasets

### 2.11 Icon Handling
- Embed custom point icons inside KMZ file
- Convert ArcGIS symbol library icons to PNG
- Support for picture marker symbols
- Fallback to standard KML icons when conversion not possible

---

## 3. Tool Parameters

### 3.1 Required Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| Input Layers | Feature Layer (multi-value) | One or more layers to export |
| Output File | File | Output .kml or .kmz path |

### 3.2 Optional Parameters
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| Output Format | String | KMZ | KML or KMZ |
| Include Fields | Field (multi-value) | All | Fields to include in popups |
| Folder Field | Field | None | Field to group features by |
| Altitude Mode | String | clampToGround | KML altitude mode |
| Altitude Field | Field | None | Field containing height values |
| Extrude | Boolean | False | Extrude features to ground |
| Time Field | Field | None | Field for time animation |
| Time End Field | Field | None | End time for TimeSpan |
| Export Labels | Boolean | True | Include feature labels |
| Combine Layers | Boolean | True | Merge into single KML |

---

## 4. Technical Architecture

### 4.1 File Structure
```
KML_Tool/
├── KMLExportTools.pyt            # Thin toolbox wrapper (tool UI/parameters only)
├── kml_export/                   # External Python package (referenced by .pyt)
│   ├── __init__.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── exporter.py           # Main export logic
│   │   ├── geometry.py           # Geometry conversion
│   │   └── projection.py         # Coordinate reprojection
│   ├── styling/
│   │   ├── __init__.py
│   │   ├── symbology.py          # ArcGIS symbology reader
│   │   ├── kml_styles.py         # KML style generation
│   │   └── icons.py              # Icon extraction/embedding
│   ├── content/
│   │   ├── __init__.py
│   │   ├── popups.py             # HTML popup generation
│   │   ├── folders.py            # Folder organization
│   │   └── time.py               # Time animation
│   └── utils/
│       ├── __init__.py
│       ├── kmz.py                # KMZ compression
│       └── validation.py         # Input validation
├── tests/
│   └── ...
├── prd.md                        # This document
└── README.md
```

### 4.2 Architecture Pattern
The `.pyt` toolbox file contains **only**:
- Tool class definitions with parameter definitions
- `execute()` method that imports and calls external modules

All business logic lives in the `kml_export/` package. This provides:
- Easier debugging and development
- Code reuse across multiple tools
- Unit testing without ArcGIS Pro
- Clean separation of UI and logic

### 4.3 Dependencies
**Required (included with ArcGIS Pro):**
- arcpy (ArcGIS Pro Python environment)
- xml.etree.ElementTree (standard library - KML generation)
- zipfile (standard library - KMZ creation)

**Optional (may need pip install in ArcGIS Pro Python):**
- simplekml - If preferred over raw XML (easier API)
- PIL/Pillow - For icon image processing (may already be installed)

### 4.4 KML Structure Output
```xml
<?xml version="1.0" encoding="UTF-8"?>
<kml xmlns="http://www.opengis.net/kml/2.2">
  <Document>
    <name>Layer Name</name>
    <Style id="style1">...</Style>
    <Folder>
      <name>Category 1</name>
      <Placemark>
        <name>Feature Name</name>
        <description><![CDATA[HTML popup]]></description>
        <styleUrl>#style1</styleUrl>
        <Point>...</Point>
      </Placemark>
    </Folder>
  </Document>
</kml>
```

---

## 5. Features Exceeding XTools Pro

| Feature | XTools Pro | This Tool |
|---------|------------|-----------|
| Graduated symbology | Limited | Full support |
| Time animation | No | Yes |
| Batch multi-layer | Basic | Advanced with combine option |
| Label export | Basic | Full with styling |
| Altitude modes | Basic | All KML modes |
| Icon embedding | Limited | Full with format conversion |
| HTML popups | Basic table | Customizable HTML |
| Progress feedback | Minimal | Detailed progress |

---

## 6. Error Handling

- Validate layer geometry types
- Handle empty layers gracefully
- Report unsupported symbology types
- Log coordinate transformation issues
- Provide meaningful error messages for common issues

---

## 7. Performance Considerations

- Stream large datasets to avoid memory issues
- Efficient geometry iteration with arcpy.da.SearchCursor
- Batch icon extraction
- Progress updates every N features

---

## 8. Future Enhancements (Out of Scope for v1.0)

- Network Links for dynamic refresh
- Raster/imagery layer support
- Region-based LOD (Level of Detail)
- Custom HTML templates
- KML import tool

---

## 9. Verification/Testing Plan

1. **Unit Tests**: Test each module independently
2. **Integration Tests**: End-to-end export with sample layers
3. **Visual Verification**: Open output in Google Earth Pro
4. **Test Datasets**:
   - Points with various symbology
   - Lines with different styles
   - Polygons with graduated colors
   - Time-enabled data
   - 3D data with Z values
