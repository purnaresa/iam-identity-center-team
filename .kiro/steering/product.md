# Product: TEAM (Temporary Elevated Access Management)

TEAM is an open-source application that integrates with AWS IAM Identity Center to manage time-bound elevated access to multi-account AWS environments.

## Core Workflow
1. Users request temporary access to an AWS account with a specific permission set
2. Requests go through an approval workflow (configurable — can be auto-approved based on eligibility)
3. Approved requests are scheduled and access is granted via IAM Identity Center account assignments
4. Access is automatically revoked when the time period elapses

## Key Concepts
- **Eligibility policies**: Define which users/groups can request which accounts and permission sets, and whether approval is required
- **Approvers**: Configured per account or OU; members of IdC groups who can approve/reject requests
- **Sessions**: Active access grants with start/end times, tracked via CloudTrail Lake
- **Settings**: Global configuration for duration limits, approval requirements, notification channels (SES, SNS, Slack)

## User Roles
- **Requester**: Any authenticated user who submits access requests
- **Approver**: Users in designated IdC groups who approve/reject requests
- **Admin**: Manages eligibility policies, approvers, and application settings
- **Auditor**: Read-only access to all requests and sessions for compliance

## Notifications
Supports SES email, SNS, and Slack notifications for request lifecycle events (pending, approved, rejected, granted, ended, error).
