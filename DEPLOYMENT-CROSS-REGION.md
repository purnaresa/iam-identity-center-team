# Cross-Region Deployment Guide
> Amplify (UI + backend) in **ap-southeast-1 (Singapore)** · IAM Identity Center in **ap-southeast-3 (Jakarta)**

## Authentication Flow

```
1. User clicks "Federated Sign In" on TEAM app
   Auth.federatedSignIn({ provider: "IAMIdentityCenter" })

2. Browser → Cognito Hosted UI (Singapore)
   GET /authorize?identity_provider=IAMIdentityCenter&...

3. Cognito → IDC (Jakarta) — SAML AuthnRequest
   Redirect to https://portal.sso.ap-southeast-3.amazonaws.com/saml/assertion/...

4. IDC authenticates user
   User logs in at msi-tsel-demo.awsapps.com

5. IDC → Cognito — SAML Response (POST /saml2/idpresponse)
   SAMLResponse contains:
   - NameID: <userName> (e.g. aws-tedy), format=persistent
   - Attribute: email = tirtawid@amazon.id

6. Cognito validates SAML assertion
   - Maps SAML attribute "email" → Cognito "email" attribute
   - Creates Cognito user: iamidentitycenter_<email>
   - Triggers PreTokenGeneration Lambda

7. PreTokenGeneration Lambda (ap-southeast-1) → IDC Identity Store (ap-southeast-3)
   - Extracts email from Cognito username: iamidentitycenter_tirtawid@amazon.id → tirtawid@amazon.id
   - Looks up IDC UserId by emails.value = tirtawid@amazon.id
   - Gets group memberships for that UserId
   - Injects groups (Admin/Auditors) into JWT claims

8. Cognito → App — authorization code
   Redirect to https://main.<app-id>.amplifyapp.com/?code=xxx

9. App exchanges code for JWT tokens
   POST /oauth2/token

10. User is logged in with correct group claims
```

### Key Cross-Region Notes
- Cognito User Pool: **ap-southeast-1** (Singapore)
- IAM Identity Center: **ap-southeast-3** (Jakarta)
- Lambda uses `IDC_REGION=ap-southeast-3` env var to call IDC APIs cross-region
- Cognito username format after SAML: `iamidentitycenter_<email>` — email is used (not IDC userName) because that's what the SAML `email` attribute contains
- IDC user lookup uses `emails.value` path (not `userName`) since the email passed is not the IDC login name

---

## Pre-requisites
- AWS CLI configured with profiles for Org Management account and TEAM account
- Node.js + Amplify CLI: `npm i -g @aws-amplify/cli@14.0.0`
- `git-remote-codecommit`: `pip install git-remote-codecommit`

---

## What Was Modified from the Original Repo

Original repo: https://github.com/aws-samples/iam-identity-center-team  
Original assumes Amplify and IAM Identity Center are in the **same region**. The following changes enable cross-region deployment.

### Files Added
| File | Purpose |
|------|---------|
| `amplify/backend/function/teamIdcProxy/teamIdcProxy-cloudformation-template.json` | CFN template for the IDC proxy Lambda |
| `amplify/backend/function/teamIdcProxy/src/index.py` | Lambda handler — proxies IDC API calls to Jakarta region |
| `deployment/parameters-template.sh` | Template for deployment parameters (added `IDC_REGION`, `INSTANCE_ARN`, `IDENTITY_STORE_ID`) |
| `DEPLOYMENT-CROSS-REGION.md` | This file |

### Files Modified
| File | Change |
|------|--------|
| `deployment/template.yml` | Added `idcRegion`, `instanceArn`, `identityStoreId` CFN parameters; passed as Amplify env vars (`IDC_REGION`, `INSTANCE_ARN`, `IDENTITY_STORE_ID`); added `_CUSTOM_IMAGE: amplify:al2023`; pinned Node.js 22.17 |
| `deployment/deploy.sh` | Added `|| true` to `create-repository` and `git remote remove origin` to prevent failure if repo/remote already exists; remote URL now uses `$TEAM_ACCOUNT_PROFILE` for GRC auth |
| `amplify/backend/backend-config.json` | Registered `teamIdcProxy` function; added it as dependency in `custom/stepfunctions` and `function/teamRouter`; added `AMPLIFY_function_teamIdcProxy_deploymentBucketName` and `AMPLIFY_function_teamIdcProxy_s3Key` parameters |

---

## Step 1 — Get your Jakarta IDC details
From the [IAM Identity Center console](https://ap-southeast-3.console.aws.amazon.com/singlesignon) → Settings:
- **Instance ARN** (e.g. `arn:aws:sso:::instance/ssoins-xxxxxxxx`)
- **Identity Store ID** (e.g. `d-xxxxxxxxxx`)
- **AWS access portal URL** (e.g. `https://<your-id>.awsapps.com/start`)

---

## Step 2 — Create `deployment/parameters.sh`
```bash
cp deployment/parameters-template.sh deployment/parameters.sh
```
Fill in your values:
```bash
IDC_LOGIN_URL=https://<your-id>.awsapps.com/start   # Jakarta IDC start URL
REGION=ap-southeast-1                                # Amplify deploys here
IDC_REGION=ap-southeast-3                            # IDC is here
INSTANCE_ARN=arn:aws:sso:::instance/ssoins-xxxxxxxx  # from Step 1
IDENTITY_STORE_ID=d-xxxxxxxxxx                       # from Step 1
TEAM_ACCOUNT=<team-account-id>
ORG_MASTER_PROFILE=<org-management-account-aws-profile>
TEAM_ACCOUNT_PROFILE=<team-account-aws-profile>
TEAM_ADMIN_GROUP=<idc-group-name-for-admins>
TEAM_AUDITOR_GROUP=<idc-group-name-for-auditors>
CLOUDTRAIL_AUDIT_LOGS=read_write
SECRET_NAME=                                         # leave empty to use CodeCommit
CACHE_TTL=604800
```

> `SECRET_NAME` must be **empty** to use CodeCommit. If set, deploy.sh uses GitHub via Secrets Manager.

---

## Step 3 — Check init (delegated admin)
Verify 819693014589 is already registered as delegated admin (already done):
```bash
aws organizations list-delegated-administrators \
  --service-principal sso.amazonaws.com \
  --profile <ORG_MASTER_PROFILE> \
  --query "DelegatedAdministrators[?Id=='<TEAM_ACCOUNT>'].Id" \
  --output text
```
If empty, run `cd deployment && ./init.sh`. Otherwise skip.

---

## Step 4 — Deploy
```bash
cd deployment
./deploy.sh
```
This will:
1. Create CodeCommit repo `team-idc-app` in ap-southeast-1 (safe to re-run — uses `|| true`)
2. Add GRC remote and push code
3. Deploy `TEAM-IDC-APP` CloudFormation stack

If CodeCommit repo already exists and code is already pushed, run CFN deploy directly:
```bash
source parameters.sh
aws cloudformation deploy \
  --region ap-southeast-1 \
  --profile $TEAM_ACCOUNT_PROFILE \
  --template-file template.yml \
  --stack-name TEAM-IDC-APP \
  --parameter-overrides \
    Login=$IDC_LOGIN_URL \
    CloudTrailAuditLogs=$CLOUDTRAIL_AUDIT_LOGS \
    teamAdminGroup="$TEAM_ADMIN_GROUP" \
    teamAuditGroup="$TEAM_AUDITOR_GROUP" \
    teamAccount=$TEAM_ACCOUNT \
    cacheTTL=$CACHE_TTL \
    idcRegion=$IDC_REGION \
    instanceArn=$INSTANCE_ARN \
    identityStoreId=$IDENTITY_STORE_ID \
  --no-fail-on-empty-changeset \
  --capabilities CAPABILITY_NAMED_IAM
```

---

## Step 5 — Monitor Amplify build
Go to the [Amplify console](https://ap-southeast-1.console.aws.amazon.com/amplify) → `TEAM-IDC-APP`. First build takes ~15–20 min.

Check build logs via CLI:
```bash
aws amplify list-jobs --app-id <APP_ID> --branch-name main \
  --region ap-southeast-1 --profile <TEAM_ACCOUNT_PROFILE>
```

---

## Step 6 — Get the app URL and configure IDC
```bash
aws cloudformation describe-stacks --stack-name TEAM-IDC-APP \
  --region ap-southeast-1 \
  --profile <TEAM_ACCOUNT_PROFILE> \
  --query "Stacks[0].Outputs[?OutputKey=='DefaultDomain'].OutputValue" \
  --output text
```
Add the returned Amplify URL as a trusted callback URL in your Jakarta IDC OIDC/SAML application config.

---

## Troubleshooting

### `Parameters: [functionteamIdcProxyArn] must have values`
`teamIdcProxy` was not registered in `backend-config.json`. Fixed by adding:
- `teamIdcProxy` function entry in `"function"` section
- `teamIdcProxy` as dependency in `custom/stepfunctions` and `function/teamRouter`
- `AMPLIFY_function_teamIdcProxy_deploymentBucketName` and `AMPLIFY_function_teamIdcProxy_s3Key` in `"parameters"` section

### `AmplifyApp CREATE_FAILED — GitHub 404`
Amplify couldn't register a webhook on the GitHub repo. Cause: expired/invalid PAT or missing `admin:repo_hook` scope.
Fix: use CodeCommit instead — set `SECRET_NAME=` (empty) in `parameters.sh`.

### `git remote remove origin — No such remote`
Harmless error. Fixed in `deploy.sh` with `2>/dev/null || true`.

### `aws codecommit create-repository — RepositoryNameExistsException`
Harmless if repo already exists. Fixed in `deploy.sh` with `|| true`.

### Cognito + IDC SAML Federation Setup

After Amplify build completes, run `integration.sh` to get the SAML values, then:

**1. Create SAML application in IDC (Jakarta) — manual console step**

Go to IAM Identity Center ap-southeast-3 → Applications → Add custom SAML 2.0 app:
- ACS URL: `https://<cognito-domain>.auth.ap-southeast-1.amazoncognito.com/saml2/idpresponse`
- SP Entity ID: `urn:amazon:cognito:sp:<user-pool-id>`
- Application start URL: output from `integration.sh`

Attribute mappings (exact values required):
| Application attribute | Maps to | Format |
|----------------------|---------|--------|
| `Subject` | `${user:subject}` | persistent |
| `email` | `${user:email}` | basic |

> ⚠️ The attribute name must be `email` (lowercase). IDC sends it as `email` in the SAML assertion.

Assign groups: `MSI-TEAM-Administrator`, `MSI-TEAM-Auditor`, `MSI-TEAM-Approver` (and any other groups).

**2. Add IDC as SAML IdP in Cognito**

Get the SAML metadata URL from IDC console → Applications → TEAM → Configuration tab.

```bash
aws cognito-idp create-identity-provider \
  --user-pool-id <user-pool-id> \
  --provider-name "IAMIdentityCenter" \
  --provider-type SAML \
  --provider-details MetadataURL="<IDC_SAML_METADATA_URL>" \
  --attribute-mapping "{\"email\":\"email\"}" \
  --region ap-southeast-1 \
  --profile <TEAM_ACCOUNT_PROFILE>
```

> ⚠️ Attribute mapping must be `"email":"email"` (both lowercase). Capital `E` or plural `emails` will fail for some users.

**3. Enable IdP on Cognito App Client**

```bash
aws cognito-idp update-user-pool-client \
  --user-pool-id <user-pool-id> \
  --client-id <web-client-id> \
  --supported-identity-providers "IAMIdentityCenter" "COGNITO" \
  --allowed-o-auth-flows "code" \
  --allowed-o-auth-scopes "aws.cognito.signin.user.admin" "email" "openid" "phone" "profile" \
  --allowed-o-auth-flows-user-pool-client \
  --callback-urls "https://main.<app-id>.amplifyapp.com/" \
  --logout-urls "https://main.<app-id>.amplifyapp.com/" \
  --region ap-southeast-1 \
  --profile <TEAM_ACCOUNT_PROFILE>
```

**4. Fix App.js to redirect directly to IDC**

In `src/App.js`, change:
```js
onClick={() => Auth.federatedSignIn()}
// to:
onClick={() => Auth.federatedSignIn({ provider: "IAMIdentityCenter" })}
```

**5. Fix PreTokenGeneration Lambda for cross-region user lookup**

The Lambda receives `iamidentitycenter_<email>` as the Cognito username. It must look up the IDC user by email (not userName) since the SAML Subject is `aws-tedy` but Cognito username suffix is the email.

In `amplify/backend/function/team06dbb7fcPreTokenGeneration/src/index.py`, replace `get_user()` with a version that tries `userName` first then falls back to `emails.value`:

```python
def get_user(username):
    client = boto3.client('identitystore', region_name=idc_region)
    try:
        response = client.get_user_id(
            IdentityStoreId=sso_instance,
            AlternateIdentifier={'UniqueAttribute': {'AttributePath': 'userName', 'AttributeValue': username}}
        )
        return response['UserId']
    except Exception:
        pass
    try:
        response = client.get_user_id(
            IdentityStoreId=sso_instance,
            AlternateIdentifier={'UniqueAttribute': {'AttributePath': 'emails.value', 'AttributeValue': username}}
        )
        return response['UserId']
    except Exception as e:
        print(f"email lookup failed: {e}")
    return None
```

After changes, push to CodeCommit to trigger Amplify rebuild, or deploy Lambda directly:
```bash
cd amplify/backend/function/team06dbb7fcPreTokenGeneration/src
zip /tmp/lambda.zip index.py
aws lambda update-function-code \
  --function-name team06dbb7fcPreTokenGeneration-main \
  --zip-file fileb:///tmp/lambda.zip \
  --region ap-southeast-1 \
  --profile <TEAM_ACCOUNT_PROFILE>
```
