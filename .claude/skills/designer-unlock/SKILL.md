---
name: designer-unlock
description: Unlock all devices across all platforms in Designer toolbox
disable-model-invocation: true
---

# Designer Unlock — Universal Toolbox Unlock

Makes all devices available across all product platforms in the Designer toolbox. Adds HW platform bits to all families/overrides, adds cross-platform models to the HW ProductMasterList, and adds HW LinkTypes to all link nodes.

**CRITICAL**: Both SQLMODELINFO.MDF and SQLREFERENCEINFO.MDF reset on every Designer restart. Run this AFTER Designer starts, BEFORE opening a project (at the "remote services" prompt).

Run all 4 queries using `mcp__designer-db__query`. All are additive and idempotent.

## Gate 1: LinkTypes (SQLMODELINFO)

Add HW LinkTypes 32/34/36 to every link node that lacks them.

```sql
INSERT INTO [C:\PROGRAMDATA\LUTRON\LUTRONELECTRONICS.LUTRONDESIGNERGAMMA_HB4QHWKZQ4PCY\ALEX\LUTRON DESIGNER 26.1.0.112\SQLMODELINFO.MDF].dbo.TBLLINKNODEINFOLINKTYPEMAP (LINKNODEINFOID, LINKTYPEID, SORTORDER)
SELECT DISTINCT lnm.LINKNODEINFOID, lt.LINKTYPEID, 1
FROM [C:\PROGRAMDATA\LUTRON\LUTRONELECTRONICS.LUTRONDESIGNERGAMMA_HB4QHWKZQ4PCY\ALEX\LUTRON DESIGNER 26.1.0.112\SQLMODELINFO.MDF].dbo.TBLLINKNODEINFOMODELINFOMAP lnm
CROSS JOIN (SELECT 32 as LINKTYPEID UNION SELECT 34 UNION SELECT 36) lt
WHERE NOT EXISTS (
  SELECT 1 FROM [C:\PROGRAMDATA\LUTRON\LUTRONELECTRONICS.LUTRONDESIGNERGAMMA_HB4QHWKZQ4PCY\ALEX\LUTRON DESIGNER 26.1.0.112\SQLMODELINFO.MDF].dbo.TBLLINKNODEINFOLINKTYPEMAP x
  WHERE x.LINKNODEINFOID = lnm.LINKNODEINFOID AND x.LINKTYPEID = lt.LINKTYPEID
);
```

## Gate 2: ProductMasterList (SQLREFERENCEINFO)

Add models to the HW ProductMasterList (cat 18) that exist in at least one OTHER
ProductMasterList but not in HW. This ensures we only add models with valid UI
resources — adding raw TBLMODELINFO entries crashes Designer on invalid string IDs.

```sql
INSERT INTO [C:\PROGRAMDATA\LUTRON\LUTRONELECTRONICS.LUTRONDESIGNERGAMMA_HB4QHWKZQ4PCY\ALEX\LUTRON DESIGNER 26.1.0.112\SQLREFERENCEINFO.MDF].dbo.TBLPRODINFOMDLINFOLISTMDLINFOID
  (PRODUCTINFOMODELINFOLISTCATID, MODELINFOID, SORTORDER, ISOBSOLETE, OVERRIDEFILTERINFOID)
SELECT 18, src.MODELINFOID,
  ROW_NUMBER() OVER (ORDER BY src.MODELINFOID) +
  (SELECT ISNULL(MAX(SORTORDER), 0) FROM [C:\PROGRAMDATA\LUTRON\LUTRONELECTRONICS.LUTRONDESIGNERGAMMA_HB4QHWKZQ4PCY\ALEX\LUTRON DESIGNER 26.1.0.112\SQLREFERENCEINFO.MDF].dbo.TBLPRODINFOMDLINFOLISTMDLINFOID WHERE PRODUCTINFOMODELINFOLISTCATID = 18),
  0, NULL
FROM (
  SELECT DISTINCT MODELINFOID
  FROM [C:\PROGRAMDATA\LUTRON\LUTRONELECTRONICS.LUTRONDESIGNERGAMMA_HB4QHWKZQ4PCY\ALEX\LUTRON DESIGNER 26.1.0.112\SQLREFERENCEINFO.MDF].dbo.TBLPRODINFOMDLINFOLISTMDLINFOID
  WHERE PRODUCTINFOMODELINFOLISTCATID <> 18
) src
WHERE src.MODELINFOID NOT IN (
  SELECT p.MODELINFOID FROM [C:\PROGRAMDATA\LUTRON\LUTRONELECTRONICS.LUTRONDESIGNERGAMMA_HB4QHWKZQ4PCY\ALEX\LUTRON DESIGNER 26.1.0.112\SQLREFERENCEINFO.MDF].dbo.TBLPRODINFOMDLINFOLISTMDLINFOID p
  WHERE p.PRODUCTINFOMODELINFOLISTCATID = 18
);
```

## Gate 3a: Family-Level ToolboxPlatformTypes (SQLMODELINFO)

OR in HW bits (4128 = HomeworksQS + myRoomLegacy) on ALL families.

```sql
UPDATE [C:\PROGRAMDATA\LUTRON\LUTRONELECTRONICS.LUTRONDESIGNERGAMMA_HB4QHWKZQ4PCY\ALEX\LUTRON DESIGNER 26.1.0.112\SQLMODELINFO.MDF].dbo.TBLFAMILYCATEGORYINFO
SET TOOLBOXPLATFORMTYPES = TOOLBOXPLATFORMTYPES | 4128
WHERE TOOLBOXPLATFORMTYPES & 4128 <> 4128;
```

## Gate 3b: Model-Level ToolboxPlatformTypes Override (SQLMODELINFO)

OR in HW bits on ALL model-level overrides. This table is checked FIRST by
`ProductInfoHelper.GetLocationBasedToolboxPlatformTypesForModel()`, before
falling back to family. Without this, models with overrides are invisible
even when the family value is correct.

```sql
UPDATE [C:\PROGRAMDATA\LUTRON\LUTRONELECTRONICS.LUTRONDESIGNERGAMMA_HB4QHWKZQ4PCY\ALEX\LUTRON DESIGNER 26.1.0.112\SQLMODELINFO.MDF].dbo.TBLMODELINFOTOOLBOXPLATFORMTYPEMAP
SET TOOLBOXPLATFORMTYPEID = TOOLBOXPLATFORMTYPEID | 4128
WHERE TOOLBOXPLATFORMTYPEID & 4128 <> 4128;
```

## After Applying

1. Click **"Leave remote services disabled"**
2. All cross-platform devices should now appear in the HW toolbox
3. Report row counts for each gate

## Notes

- All queries are **additive** (`|`, `INSERT ... NOT IN`) — never delete or overwrite
- All queries are **idempotent** — safe to run multiple times
- `4128 = 0x1020 = HomeworksQS (0x20) + myRoomLegacy (0x1000)`
- Gate 2 scopes to models in other ProductMasterLists to avoid crash on models with invalid string IDs in visual 9 (the HW visual context)
- Gate 4 (DeviceClass comparison at pairing) has no DB fix — pair using the RA3 model, then swap via serial injection if needed

### Model Prefix Reference

| RA3 | HW Equivalent | Type |
|-----|---------------|------|
| RRD | HQR/HQRD | Dimmers/switches (CCA) |
| RRT | HQRT | seeTouch keypads (CCA) |
| RRST | HRST | Sunnata (Thread) |
| RRDE | — | Extension dimmers (CCA) |
| RRL | — | Newer models |
