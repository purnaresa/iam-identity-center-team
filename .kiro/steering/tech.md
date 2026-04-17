# Tech Stack

## Frontend
- React 17 (Create React App via `react-scripts`)
- JavaScript (no TypeScript)
- UI libraries: AWS Cloudscape (`@awsui/components-react`) and Ant Design (`antd`)
- Routing: `react-router-dom` v5 (`BrowserRouter`, `Switch`, `Route`)
- Forms: `react-hook-form` with `@hookform/resolvers`
- Auth: `aws-amplify` v5 with Cognito federated sign-in via IAM Identity Center
- GraphQL client: `@aws-amplify/api` with auto-generated queries/mutations/subscriptions in `src/graphql/`
- Styling: SASS, CSS

## Backend
- AWS Amplify (CLI v14) for infrastructure orchestration
- AWS AppSync (GraphQL API) with Cognito User Pools + IAM auth
- DynamoDB tables (auto-managed by Amplify `@model` directive)
- AWS Lambda functions written in Python (various runtimes)
  - Python dependencies managed via Pipfile/Pipenv
  - Lambda functions use `boto3` for AWS SDK calls
  - AppSync mutations from Lambda use `requests` with `requests_aws_sign` (IAM SigV4)
- AWS Step Functions for request lifecycle workflows (grant, revoke, reject, schedule, approval)
- VTL resolvers for AppSync custom logic
- CloudTrail Lake for audit logging

## Infrastructure / Deployment
- AWS CloudFormation (`deployment/template.yml`) deploys the Amplify app and triggers builds
- Amplify Hosting (manual type) with CodeCommit or external repo
- Deployment scripts in `deployment/` (bash): `deploy.sh`, `destroy.sh`, `update.sh`, `init.sh`
- Parameters configured via `deployment/parameters-template.sh`

## Common Commands

```bash
# Install frontend dependencies
npm ci

# Start local dev server
npm start

# Production build
npm run build

# Run tests
npm test

# Deploy (from deployment/ directory, after configuring parameters.sh)
cd deployment && bash deploy.sh
```

## GraphQL Schema
- Source of truth: `amplify/backend/api/team/schema.graphql`
- Auto-generated client code: `src/graphql/queries.js`, `mutations.js`, `subscriptions.js`
- Schema uses Amplify directives: `@model`, `@auth`, `@function`, `@index`, `@ttl`
