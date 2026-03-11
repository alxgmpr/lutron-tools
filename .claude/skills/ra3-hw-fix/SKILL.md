---
name: ra3-hw-fix
description: "[DEPRECATED — use /designer-unlock] Quick Gate 3a patch for opening HW projects with RA3 devices"
disable-model-invocation: true
---

# RA3 HW Fix (Deprecated)

**This skill is superseded by `/designer-unlock`** which applies all 4 gates (LinkTypes, ProductMasterList, family ToolboxPlatformTypes, AND model-level ToolboxPlatformTypes override).

This skill only patches Gate 3a (family-level ToolboxPlatformTypes), which is insufficient — Gate 3b (model-level overrides in `TBLMODELINFOTOOLBOXPLATFORMTYPEMAP`) also needs patching for 10 models, and Gates 1-2 are needed for toolbox visibility and pairing.

Use `/designer-unlock` instead.
