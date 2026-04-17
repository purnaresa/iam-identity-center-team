# Project Structure

```
├── src/                          # React frontend
│   ├── App.js                    # Root component, auth flow
│   ├── index.js                  # Entry point
│   ├── parameters.json           # Runtime config
│   ├── components/
│   │   ├── Admin/                # Admin views (Approvers, Eligible, Settings)
│   │   ├── Approvals/            # Approval workflow UI
│   │   ├── Audit/                # Auditor views (AuditApprovals, AuditSessions)
│   │   ├── Navigation/           # Nav, Header, Landing, routing
│   │   ├── Requests/             # Request creation and viewing
│   │   ├── Sessions/             # Active sessions and session audit
│   │   └── Shared/               # Reusable components
│   ├── graphql/                  # Auto-generated GraphQL operations (do not edit manually)
│   │   ├── queries.js
│   │   ├── mutations.js
│   │   ├── subscriptions.js
│   │   └── schema.json
│   └── models/                   # Auto-generated Amplify DataStore models
│
├── amplify/backend/              # Amplify backend resources
│   ├── api/team/
│   │   ├── schema.graphql        # GraphQL schema (source of truth)
│   │   └── resolvers/            # Custom VTL resolvers
│   ├── auth/                     # Cognito user pool config
│   ├── function/                 # Lambda functions (Python)
│   │   ├── teamRouter/           # Main event router (DynamoDB stream handler)
│   │   ├── teamNotifications/    # SES/SNS/Slack notification sender
│   │   ├── teamIdcProxy/         # IAM Identity Center operations proxy
│   │   ├── teamStatus/           # Request status updater
│   │   ├── teamGetPermissionSets/# Permission set fetcher
│   │   ├── teamListGroups/       # IdC group membership resolver
│   │   ├── teamgetAccounts/      # AWS Organizations account lister
│   │   ├── teamgetOUs/           # OU hierarchy fetcher
│   │   ├── teamgetOUAccounts/    # OU-to-accounts resolver with caching
│   │   ├── teamgetUserPolicy/    # User eligibility policy resolver
│   │   ├── teamgetEntitlement/   # Entitlement lookup
│   │   ├── teamvalidateRequest/  # Request validation
│   │   ├── teaminvalidateOUCache/# OU cache invalidation
│   │   └── ...                   # Other Lambda functions
│   └── custom/                   # Custom CloudFormation resources
│       ├── stepfunctions/        # Step Functions state machines
│       ├── sns/                  # SNS notification topic
│       ├── cloudtrailLake/       # CloudTrail Lake event data store
│       └── s3bucketSecurity/     # S3 bucket security policies
│
├── deployment/                   # Deployment scripts and CloudFormation
│   ├── template.yml              # Main CFN template (Amplify app)
│   ├── deploy.sh                 # Deploy script
│   ├── destroy.sh                # Teardown script
│   ├── update.sh                 # Update script
│   └── parameters-template.sh    # Deployment parameter template
│
├── docs/                         # Jekyll documentation site
├── public/                       # Static assets (index.html, logo, etc.)
└── parameters.js                 # Build-time parameter injection for Amplify
```

## Key Conventions
- Frontend components are organized by feature domain (Admin, Requests, Approvals, Sessions, Audit)
- Lambda functions each live in their own directory under `amplify/backend/function/`
- Lambda function entry points are always `src/index.py` with a `handler` (or `lambda_handler`) function
- GraphQL files in `src/graphql/` are auto-generated from the schema — edit `schema.graphql` instead
- Infrastructure changes go through Amplify CLI or the CloudFormation templates in `deployment/`
