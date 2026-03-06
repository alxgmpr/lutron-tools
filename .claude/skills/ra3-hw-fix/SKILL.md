---
name: ra3-hw-fix
description: Patch Designer SQLMODELINFO to allow RA3 devices in Homeworks projects
disable-model-invocation: true
---

# RA3 HW Fix

Patches the SQLMODELINFO database so Designer doesn't block opening HW projects containing RA3 devices ("Your account is not able to open this project").

## Instructions

Run this SQL using the `designer-db` MCP tool (`mcp__designer-db__query`):

```sql
USE [C:\PROGRAMDATA\LUTRON\LUTRONELECTRONICS.LUTRONDESIGNERGAMMA_HB4QHWKZQ4PCY\ALEX\LUTRON DESIGNER 26.0.2.100\SQLMODELINFO.MDF];

UPDATE TBLFAMILYCATEGORYINFO
SET TOOLBOXPLATFORMTYPES = TOOLBOXPLATFORMTYPES | 4128
FROM TBLFAMILYCATEGORYINFO f
JOIN TBLFAMILYCATINFOMODELINFOMAP fm ON fm.FAMILYCATEGORYINFOID = f.FAMILYCATEGORYINFOID
JOIN TBLMODELINFO m ON m.MODELINFOID = fm.MODELINFOID
WHERE m.LUTRONMODELNUMBERBASE LIKE 'RR%'
AND f.TOOLBOXPLATFORMTYPES & 4128 = 0;
```

Report the number of rows affected. If 0 rows affected, tell the user the patch was already applied. Remind them to click "Leave remote services disabled" in Designer to proceed.
