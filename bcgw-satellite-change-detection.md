# BCGW "Satellite Imagery - Change Detection"

This note explains how the BCGW `DATA_SOURCE` value `Satellite Imagery - Change Detection`
is handled during Consolidated Cutblocks layer creation. The logic is implemented in
`CC_Step_1_create_layer.py` under the "BCGW Consolidated Cutblocks processing" section.

## Where It Starts

BCGW consolidated cutblocks are filtered to keep:

- `DATA_SOURCE` in `('VRI', 'Satellite Imagery - Change Detection')`
- `HARVEST_START_DATE` or `HARVEST_END_DATE` is not null

After selection, fields are prefixed with `BCGW_CC_` (for example `DATA_SOURCE` becomes
`BCGW_CC_DATA_SOURCE`).

## Flowchart

```mermaid
flowchart TD
    A[BCGW consolidated cutblocks] --> B{DATA_SOURCE in<br/>VRI or Satellite?}
    B -- No --> Z[Exclude]
    B -- Yes --> C{Has harvest start or end date?}
    C -- No --> Z
    C -- Yes --> D[Select to in-memory<br/>BCGW_CC_* fields]
    D --> E{BCGW_CC_DATA_SOURCE<br/>contains "Satellite"?}
    E -- Yes --> F[process_landsat_overlay:<br/>exclude >= 75% overlap<br/>with protected layers]
    E -- No --> G[VRI blocks kept]
    F --> H[Merge VRI + filtered Satellite]
    G --> H
    H --> I[Remove internal overlaps]
    I --> J[De-duplicate vs Results/BCTS/FTEN<br/>by overlap + date match]
    J --> K[Merge into final layer]
    K --> L[CC_BLOCK_CODE fallback<br/>uses BCGW_CC_DATA_SOURCE]
    K --> M[CC_REFERENCE_YEAR uses<br/>BCGW_CC_DATA_SOURCE_DATE when CC_SOURCE=BCGW_CC]
```

## Satellite-Specific Rules

- A record is considered "satellite" when `BCGW_CC_DATA_SOURCE LIKE '%Satellite%'`.
- Satellite records are filtered against protected/ownership layers. Any block with
  **>= 75% overlap** is excluded.
- VRI blocks are kept as-is and merged with the filtered satellite set.

## Examples

### Example 1: Satellite block kept

Input (BCGW consolidated cutblocks):

- `DATA_SOURCE` = `Satellite Imagery - Change Detection`
- `HARVEST_START_DATE` = `2018-06-12`
- Overlap with protected layers = `32%`

Result:

- Passes initial filter (source + harvest date)
- Treated as satellite
- Kept because overlap < 75%
- Merged into BCGW CC blocks and then into the final layer
- `CC_BLOCK_CODE` uses `BCGW_CC_DATA_SOURCE` if other sources are null
- `CC_REFERENCE_YEAR` uses `BCGW_CC_DATA_SOURCE_DATE` if `CC_SOURCE = 'BCGW_CC'`

### Example 2: Satellite block excluded

Input:

- `DATA_SOURCE` = `Satellite Imagery - Change Detection`
- `HARVEST_END_DATE` = `2017-09-01`
- Overlap with protected layers = `84%`

Result:

- Passes initial filter
- Treated as satellite
- Excluded because overlap >= 75%
- Does not proceed to merge or final layer

### Example 3: VRI block retained

Input:

- `DATA_SOURCE` = `VRI`
- `HARVEST_START_DATE` = `2014-07-03`

Result:

- Passes initial filter
- Not treated as satellite
- Kept without the protected-area overlap filter
- Merged with filtered satellite blocks

## Outputs Affected

Once merged into the final layer, BCGW CC records affect:

- `CC_BLOCK_CODE` (uses `BCGW_CC_DATA_SOURCE` as a fallback)
- `CC_REFERENCE_YEAR` (uses `BCGW_CC_DATA_SOURCE_DATE` when `CC_SOURCE = 'BCGW_CC'`)

