# Requirements Document

## Introduction

The TEAM (Temporary Elevated Access Management) solution is an AWS Amplify application that integrates with IAM Identity Center (IDC) to manage temporary elevated access to AWS accounts. The original TEAM codebase assumes that Amplify and IDC are deployed in the same AWS region. All Lambda functions create IDC API clients without specifying a region, SSO instance details are discovered at runtime via `ListInstances` in the local region, Step Functions call `sso-admin` APIs directly, and the Cognito username format is predictable because Cognito and IDC are co-located.

This feature enables cross-region operation where the Amplify application (including Cognito, Lambda, Step Functions, and DynamoDB) runs in one region while IDC runs in a different region. This is necessary when Amplify is not available in the IDC region (e.g., Amplify in ap-southeast-1 Singapore, IDC in ap-southeast-3 Jakarta).

## Glossary

- **TEAM_Application**: The Temporary Elevated Access Management Amplify application, including its frontend, backend Lambda functions, Step Functions, DynamoDB tables, and Cognito User Pool
- **IDC**: AWS IAM Identity Center, the identity provider service
- **Amplify_Region**: The AWS region where the Amplify application and all its backend resources are deployed
- **IDC_Region**: The AWS region where IAM Identity Center and its Identity Store are deployed
- **Identity_Store**: The IDC Identity Store service that stores user and group data, accessed via the `identitystore` API
- **SSO_Admin**: The IDC administrative API (`sso-admin`) used for managing SSO instances, permission sets, and account assignments
- **Cognito_User_Pool**: The Amazon Cognito User Pool in the Amplify_Region that authenticates users via SAML federation with IDC
- **PreTokenGeneration_Lambda**: The Cognito trigger Lambda (`team06dbb7fcPreTokenGeneration`) that enriches JWT tokens with IDC group membership claims
- **TeamRouter_Lambda**: The DynamoDB stream-triggered Lambda (`teamRouter`) that orchestrates access request workflows
- **Grant_State_Machine**: The Step Functions state machine that creates temporary account assignments via `sso-admin:CreateAccountAssignment`
- **Revoke_State_Machine**: The Step Functions state machine that removes account assignments via `sso-admin:DeleteAccountAssignment`
- **Deployment_Template**: The CloudFormation template (`deployment/template.yml`) that creates the Amplify app and triggers the initial build

## Requirements

### Requirement 1: Cross-Region IDC API Access

**User Story:** As a TEAM administrator, I want all backend components that call IDC APIs to target the correct IDC region, so that the application functions when Amplify and IDC are in different AWS regions.

#### Acceptance Criteria

1. WHEN a Lambda function creates a boto3 client for the `identitystore` service, THE TEAM_Application SHALL use a configurable IDC_Region as the `region_name` parameter instead of defaulting to the Lambda execution region
2. WHEN a Lambda function creates a boto3 client for the `sso-admin` service, THE TEAM_Application SHALL use the same configurable IDC_Region as the `region_name` parameter instead of defaulting to the Lambda execution region
3. THE TEAM_Application SHALL apply cross-region client configuration to all Lambda functions that call IDC APIs: PreTokenGeneration_Lambda, TeamRouter_Lambda, teamGetPermissionSets, teamListGroups, teamgetUsers, teamgetIdCGroups, and teamgetMgmtAccountDetails

### Requirement 2: Explicit SSO Instance Configuration

**User Story:** As a TEAM administrator, I want to provide the SSO instance ARN and Identity Store ID as deployment-time configuration, so that Lambda functions do not rely on `sso-admin:ListInstances` which returns empty results when called from a region where IDC is not deployed.

#### Acceptance Criteria

1. WHEN the SSO instance ARN and Identity Store ID are provided as configuration, THE TEAM_Application SHALL use those values directly without calling `sso-admin:ListInstances`
2. WHEN the SSO instance ARN or Identity Store ID is not provided, THE TEAM_Application SHALL fall back to calling `sso-admin:ListInstances` in the IDC_Region to discover the SSO instance
3. THE TEAM_Application SHALL apply this configuration-first lookup to all Lambda functions that require SSO instance details: PreTokenGeneration_Lambda, TeamRouter_Lambda, teamGetPermissionSets, teamListGroups, teamgetUsers, teamgetIdCGroups, and teamgetMgmtAccountDetails

### Requirement 3: Cross-Region SAML Username Resolution

**User Story:** As a TEAM user authenticating via SAML federation, I want the application to correctly resolve my identity in IDC regardless of the Cognito username format, so that my group memberships and entitlements are applied when Cognito and IDC are in different regions.

#### Acceptance Criteria

1. WHEN the PreTokenGeneration_Lambda receives a Cognito event with a federated username, THE PreTokenGeneration_Lambda SHALL extract the user identifier by splitting `event['userName']` on the first underscore delimiter and using the portion after the delimiter
2. WHEN looking up a user in the Identity_Store, THE TEAM_Application SHALL first attempt lookup by the `userName` attribute path
3. WHEN the `userName` lookup does not find a matching user, THE TEAM_Application SHALL fall back to lookup by the `emails.value` attribute path using the same identifier
4. IF both `userName` and `emails.value` lookups fail, THEN THE TEAM_Application SHALL return None and log the failure with the attempted identifier
5. THE TEAM_Application SHALL apply this dual-lookup strategy in both the PreTokenGeneration_Lambda and the TeamRouter_Lambda `get_user()` function

### Requirement 4: Cross-Region Account Assignment Operations

**User Story:** As a TEAM administrator, I want account assignment grant and revoke operations to execute against the IDC region, so that Step Functions running in the Amplify region can manage SSO account assignments in the IDC region.

#### Acceptance Criteria

1. WHEN the Grant_State_Machine needs to create an account assignment, THE TEAM_Application SHALL execute the `sso-admin:CreateAccountAssignment` API call targeting the IDC_Region rather than the Amplify_Region
2. WHEN the Revoke_State_Machine needs to delete an account assignment, THE TEAM_Application SHALL execute the `sso-admin:DeleteAccountAssignment` API call targeting the IDC_Region rather than the Amplify_Region
3. IF an account assignment operation fails due to a client error, THEN THE TEAM_Application SHALL propagate the error to the calling state machine for error handling

### Requirement 5: Frontend Direct Provider Sign-In

**User Story:** As a TEAM user, I want to be redirected directly to the IAM Identity Center SAML provider when signing in, so that I authenticate against the correct identity provider without seeing the generic Cognito hosted UI.

#### Acceptance Criteria

1. WHEN a user clicks the sign-in button, THE TEAM_Application frontend SHALL invoke `Auth.federatedSignIn` with the `provider` parameter set to the configured SAML identity provider name
2. THE TEAM_Application frontend SHALL redirect the user directly to the IDC SAML authentication endpoint without displaying the Cognito hosted UI provider selection page

### Requirement 6: Cross-Region Deployment Configuration

**User Story:** As a TEAM administrator deploying the application, I want to specify the IDC region, SSO instance ARN, and Identity Store ID as deployment parameters, so that the deployment infrastructure propagates these values to all backend components that need them.

#### Acceptance Criteria

1. THE Deployment_Template SHALL accept the IDC region, SSO instance ARN, and Identity Store ID as input parameters
2. WHEN the CloudFormation stack is deployed, THE Deployment_Template SHALL propagate the IDC region, SSO instance ARN, and Identity Store ID as Amplify branch environment variables
3. WHEN Amplify builds the backend, THE TEAM_Application SHALL pass the IDC region, SSO instance ARN, and Identity Store ID as environment variables to each Lambda function that requires IDC API access
4. THE Deployment_Template SHALL provide a sensible default for the IDC region parameter to support same-region deployments without additional configuration

### Requirement 7: Cognito-IDC SAML Federation Configuration

**User Story:** As a TEAM administrator, I want the SAML federation between Cognito and IDC to work correctly across regions, so that users can authenticate through IDC in one region and have their identity recognized by Cognito in another region.

#### Acceptance Criteria

1. WHEN configuring the Cognito identity provider, THE TEAM_Application SHALL support registering a SAML provider that points to the IDC instance in the IDC_Region
2. WHEN configuring SAML attribute mapping, THE TEAM_Application SHALL map the SAML `email` attribute to the Cognito `email` attribute so that user identity is resolved by email
3. WHEN configuring the Cognito app client, THE TEAM_Application SHALL include the SAML identity provider in the list of supported identity providers alongside the default Cognito provider
