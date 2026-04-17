# Implementation Plan: Cross-Region IDC Support

## Overview

Convert the feature design into a series of prompts for a code-generation LLM that will implement each step with incremental progress. Make sure that each prompt builds on the previous prompts, and ends with wiring things together. There should be no hanging or orphaned code that isn't integrated into a previous step. Focus ONLY on tasks that involve writing, modifying, or testing code.

This plan covers the 4 remaining gaps identified in the design's Gap Analysis. The teammate has already implemented boto3 `region_name` propagation, SSO instance env var fallback, `get_user()` dual-lookup, the Step Functions IDC proxy, frontend direct sign-in, and the CloudFormation `deployment/template.yml` parameters. The tasks below close out the remaining work:

1. Fix hardcoded `'ap-southeast-3'` defaults in all 8 Lambdas so same-region deployments work without setting `IDC_REGION`
2. Fix `teamRouter` username prefix stripping to handle cross-region usernames
3. Fix `get_approvers()` so `approver_ids` match the approver's actual Cognito username (critical for AppSync `@auth`)
4. Propagate `IDC_REGION` / `INSTANCE_ARN` / `IDENTITY_STORE_ID` through `parameters.js` and add the matching CFN template parameters to the 6 Lambdas that are still missing them

## Tasks

- [ ] 1. Fix `IDC_REGION` default fallback in all 8 Lambda functions
  - Change `os.getenv('IDC_REGION', 'ap-southeast-3')` to `os.getenv('IDC_REGION', os.environ.get('REGION'))` so the default falls back to the Lambda's own region for same-region deployments
  - Update the following files:
    - `amplify/backend/function/team06dbb7fcPreTokenGeneration/src/index.py`
    - `amplify/backend/function/teamRouter/src/index.py`
    - `amplify/backend/function/teamGetPermissionSets/src/index.py`
    - `amplify/backend/function/teamListGroups/src/index.py`
    - `amplify/backend/function/teamgetUsers/src/index.py`
    - `amplify/backend/function/teamgetIdCGroups/src/index.py`
    - `amplify/backend/function/teamgetMgmtAccountDetails/src/index.py`
    - `amplify/backend/function/teamIdcProxy/src/index.py`
  - _Requirements: 1.1, 1.2, 1.3_

- [ ] 2. Fix username prefix stripping in `teamRouter` handler
  - [ ] 2.1 Replace `(data["username"]["S"])[4:]` with `data["username"]["S"].split("_", 1)[1]` in the `handler()` function of `amplify/backend/function/teamRouter/src/index.py` so cross-region usernames (`iamidentitycenter_<email>`) are parsed correctly
    - _Requirements: 3.1, 3.5_

  - [ ]* 2.2 Write property-based test for username prefix stripping
    - **Property 1: Username prefix stripping preserves the identifier after the first underscore**
    - **Validates: Requirements 3.1**
    - Use Hypothesis to generate random strings containing at least one underscore; assert that `s.split("_", 1)[1]` equals `s[s.index("_") + 1:]`
    - Minimum 100 iterations
    - Tag: `Feature: cross-region-idc-support, Property 1: Username prefix stripping preserves the identifier after the first underscore`

- [ ] 3. Fix `get_approvers()` to construct `approver_id` matching the approver's Cognito username
  - In `amplify/backend/function/teamRouter/src/index.py`, update `get_approvers(userId)` so the returned `approver_id` matches the approver's actual Cognito username (not the hardcoded `"idc_" + response['UserName']`)
  - Look up the approver's Cognito username by email via `cognito-idp:ListUsers` with a `email = "..."` filter (reuse the existing `get_email` pattern and `cognito` client usage already present in the file)
  - Preserve the `.lower()` normalization already applied to `approver_ids` in `get_approvers_details`
  - _Requirements: 3.1, 3.5_

- [ ] 4. Propagate IDC env vars from `parameters.js` to affected Lambdas' `parameters.json`
  - [ ] 4.1 Add a new function in `parameters.js` that reads `IDC_REGION`, `INSTANCE_ARN`, `IDENTITY_STORE_ID` from `process.env` and writes them into each affected Lambda's `parameters.json`
    - Affected Lambdas: `team06dbb7fcPreTokenGeneration`, `teamGetPermissionSets`, `teamListGroups`, `teamgetUsers`, `teamgetIdCGroups`, `teamgetMgmtAccountDetails`
    - Create the `parameters.json` file for Lambdas that don't already have one
    - Invoke the new function alongside the existing `update_*_parameters()` calls at the bottom of `parameters.js`
    - _Requirements: 6.2, 6.3_

  - [ ] 4.2 Add `idcRegion` / `instanceArn` / `identityStoreId` CFN parameters and Lambda env vars to `team06dbb7fcPreTokenGeneration`
    - Edit `amplify/backend/function/team06dbb7fcPreTokenGeneration/team06dbb7fcPreTokenGeneration-cloudformation-template.json`
    - Add `idcRegion` and `identityStoreId` to `Parameters` (matching the pattern in `teamRouter-cloudformation-template.json`)
    - Add `IDC_REGION` and `IDENTITY_STORE_ID` to the Lambda `Environment.Variables` block
    - _Requirements: 6.3_

  - [ ] 4.3 Add `idcRegion` / `instanceArn` / `identityStoreId` CFN parameters and Lambda env vars to `teamGetPermissionSets`
    - Edit `amplify/backend/function/teamGetPermissionSets/teamGetPermissionSets-cloudformation-template.json`
    - Add all three parameters (`idcRegion`, `instanceArn`, `identityStoreId`) and wire them to `IDC_REGION`, `INSTANCE_ARN`, `IDENTITY_STORE_ID` env vars
    - _Requirements: 6.3_

  - [ ] 4.4 Add `idcRegion` / `identityStoreId` CFN parameters and Lambda env vars to `teamListGroups`
    - Edit `amplify/backend/function/teamListGroups/teamListGroups-cloudformation-template.json`
    - Add `idcRegion`, `identityStoreId` parameters and wire to `IDC_REGION`, `IDENTITY_STORE_ID` env vars
    - _Requirements: 6.3_

  - [ ] 4.5 Add `idcRegion` / `identityStoreId` CFN parameters and Lambda env vars to `teamgetUsers`
    - Edit `amplify/backend/function/teamgetUsers/teamgetUsers-cloudformation-template.json`
    - Add `idcRegion`, `identityStoreId` parameters and wire to `IDC_REGION`, `IDENTITY_STORE_ID` env vars
    - _Requirements: 6.3_

  - [ ] 4.6 Add `idcRegion` / `identityStoreId` CFN parameters and Lambda env vars to `teamgetIdCGroups`
    - Edit `amplify/backend/function/teamgetIdCGroups/teamgetIdCGroups-cloudformation-template.json`
    - Add `idcRegion`, `identityStoreId` parameters and wire to `IDC_REGION`, `IDENTITY_STORE_ID` env vars
    - _Requirements: 6.3_

  - [ ] 4.7 Add `idcRegion` / `instanceArn` / `identityStoreId` CFN parameters and Lambda env vars to `teamgetMgmtAccountDetails`
    - Edit `amplify/backend/function/teamgetMgmtAccountDetails/teamgetMgmtAccountDetails-cloudformation-template.json`
    - Add all three parameters and wire to `IDC_REGION`, `INSTANCE_ARN`, `IDENTITY_STORE_ID` env vars
    - _Requirements: 6.3_

- [ ] 5. Final checkpoint - Ensure all changes are consistent
  - Ensure all tests pass, ask the user if questions arise.
  - Verify that all 8 Lambdas use the `os.environ.get('REGION')` fallback pattern
  - Verify that the 6 updated Lambda CFN templates reference their new parameters in `Environment.Variables`
  - Verify `parameters.js` runs without error when `IDC_REGION` / `INSTANCE_ARN` / `IDENTITY_STORE_ID` are unset (same-region case)

## Notes

- Tasks marked with `*` are optional and can be skipped
- The property-based test (2.2) validates the only universal correctness property in the design; unit tests are not listed since no test infrastructure exists in the Lambda directories and the workspace guidance is to keep changes minimal
- Integration and deployment verification (CFN stack deploy, SAML federation smoke test) are not coding tasks and are handled outside this plan

## Workflow Complete

This workflow produces the design and planning artifacts. Begin executing tasks by opening `tasks.md` and clicking "Start task" next to each item.
