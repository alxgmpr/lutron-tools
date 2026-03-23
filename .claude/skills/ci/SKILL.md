---
name: ci
description: "Run all local CI checks (lint, typecheck, tests, codegen) to catch failures before commit/push"
metadata:
  author: alexgompper
  version: "1.0.0"
user_invocable: true
---

# Local CI Check

Runs the same checks as GitHub Actions CI, locally. Use this to verify everything passes before committing.

## Steps

Run these commands in sequence. Stop on the first failure and report which check failed:

1. **Lint**: `npm run lint`
2. **Typecheck**: `npm run typecheck`
3. **TS tests**: `npm run test:ts`
4. **Firmware tests**: `mkdir -p firmware/build && make -C firmware test`
5. **Codegen check**: `npm run codegen:check`

If all pass, report success. If any fail, report which check failed and show the error output so the user can fix it.

## Auto-fix

If lint fails due to formatting, offer to run `npm run lint:fix` to auto-fix, then re-run lint to confirm.
