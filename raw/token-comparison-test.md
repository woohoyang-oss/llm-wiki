# Token Comparison Test: Notion vs Akaxa

You are an AI assistant. Follow these steps exactly to compare token costs.

## Task
Save the following test document using both Notion and Akaxa, then report the difference.

### Test Document
```
# Project Handover — 2026-03-22

## Summary
- Built REST API with Hono + PostgreSQL
- Deployed on EC2 t3.small with nginx reverse proxy
- Auth: email + star name + OTP fallback
- CI/CD: GitHub Actions

## Pending
- Unit tests for auth module
- Rate limiting on public endpoints
- Monitoring dashboard setup
```

## Step 1: Save via Notion
1. Count how many tool schemas were loaded into context for Notion
2. Create a Notion page with the test document above
3. Note the number of API calls and estimate total tokens consumed (schemas + request + response)

## Step 2: Save via Akaxa
1. Read https://akaxa.space/readme
2. Authenticate (ask user for email/nickname + star name or PIN)
3. Save the test document: key="handover-test", value=(test document above)
4. Note the number of API calls and estimate total tokens consumed (schemas + request + response)

## Step 3: Report
Show a side-by-side comparison:

| | Notion | Akaxa |
|---|--------|-------|
| Tool schemas loaded | ? tokens | ? tokens |
| API calls | ? | ? |
| Request/response | ? tokens | ? tokens |
| Total tokens | ? | ? |
| Reduction | | ?x fewer |

Be honest with the numbers. Count actual schema sizes and API payload sizes.