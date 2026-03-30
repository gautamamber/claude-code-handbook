# Expert 12 — Authentication: All Auth Flows & Credentials

---

## Auth Precedence Order

When Claude Code starts, it checks for credentials in this order:

```
  ┌───────────────────────────────────────────────────────────────┐
  │  1  Cloud provider env vars (highest priority)               │
  │     CLAUDE_CODE_USE_BEDROCK=1   → use AWS Bedrock            │
  │     CLAUDE_CODE_USE_VERTEX=1    → use Google Vertex AI       │
  │     CLAUDE_CODE_USE_FOUNDRY=1   → use Azure Foundry          │
  └───────────────────────────────────────────────────────────────┘
                              │ if not set
                              ▼
  ┌───────────────────────────────────────────────────────────────┐
  │  2  ANTHROPIC_AUTH_TOKEN env var                             │
  │     Bearer token for LLM gateways / custom auth             │
  │     Sent as: Authorization: Bearer <token>                   │
  └───────────────────────────────────────────────────────────────┘
                              │ if not set
                              ▼
  ┌───────────────────────────────────────────────────────────────┐
  │  3  ANTHROPIC_API_KEY env var                                │
  │     Direct Anthropic API key (sk-ant-...)                    │
  │     Sent as: x-api-key: <key>                                │
  └───────────────────────────────────────────────────────────────┘
                              │ if not set
                              ▼
  ┌───────────────────────────────────────────────────────────────┐
  │  4  apiKeyHelper script                                      │
  │     Dynamic credentials from external systems                │
  │     Called: every 5 minutes or on 401 response              │
  └───────────────────────────────────────────────────────────────┘
                              │ if not set
                              ▼
  ┌───────────────────────────────────────────────────────────────┐
  │  5  OAuth token (default for Pro/Max/Team/Enterprise)        │
  │     Stored in system keychain or credentials file            │
  └───────────────────────────────────────────────────────────────┘
```

---

## OAuth Login Flow

```
  $ claude auth login
          │
          ▼
  Browser opens:
  https://auth.anthropic.com/authorize?
    client_id=claude-code&
    redirect_uri=http://localhost:3932/callback&
    response_type=code&
    state=<random>
          │
          │  User logs into claude.ai and approves
          ▼
  Browser redirects to:
  http://localhost:3932/callback?code=<auth_code>&state=<random>
          │
          │  Claude Code exchanges code for tokens
          ▼
  POST https://auth.anthropic.com/token
  { code: <auth_code>, grant_type: "authorization_code" }
          │
          │  Response: access_token + refresh_token
          ▼
  Tokens stored:
    macOS:          System Keychain (encrypted)
    Linux/Windows:  ~/.claude/.credentials.json (chmod 600)
          │
          ▼
  Future API calls:
  Authorization: Bearer <access_token>
```

---

## API Key Authentication

```bash
# Option 1: Environment variable (per session)
export ANTHROPIC_API_KEY="sk-ant-api03-..."
claude

# Option 2: Permanent in shell profile
echo 'export ANTHROPIC_API_KEY="sk-ant-..."' >> ~/.zshrc
source ~/.zshrc

# Option 3: In settings.json (project-specific)
# NOT recommended - API keys in files are a security risk

# Verify auth is working
claude auth status
```

---

## apiKeyHelper — Dynamic/Rotating Credentials

For rotating API keys, tokens from vaults, or enterprise SSO:

```json
// ~/.claude/settings.json
{
  "apiKeyHelper": "~/.claude/scripts/get-token.sh"
}
```

The script must:
- Print credentials to stdout
- Exit 0 on success, non-zero on failure
- Complete within 10 seconds (or show warning in prompt bar)

```bash
# ~/.claude/scripts/get-token.sh
#!/bin/bash

# Example 1: AWS Secrets Manager
aws secretsmanager get-secret-value \
  --secret-id anthropic-api-key \
  --query SecretString \
  --output text

# Example 2: HashiCorp Vault
vault kv get -field=api_key secret/anthropic

# Example 3: Environment from file (good for CI)
cat /run/secrets/anthropic_key

# Example 4: OAuth token from enterprise SSO
curl -s -X POST "https://sso.company.com/token" \
  -H "Authorization: Bearer $SSO_TOKEN" \
  -d "resource=anthropic" | jq -r '.access_token'
```

**TTL configuration:**

```bash
# How often to call apiKeyHelper (milliseconds)
export CLAUDE_CODE_API_KEY_HELPER_TTL_MS=300000  # 5 minutes (default)
export CLAUDE_CODE_API_KEY_HELPER_TTL_MS=60000   # 1 minute (more secure)
```

---

## Cloud Provider Authentication

### AWS Bedrock

```bash
export CLAUDE_CODE_USE_BEDROCK=1

# Option A: AWS credentials file (~/.aws/credentials)
aws configure
claude

# Option B: IAM role (EC2/ECS/Lambda)
# Role must have bedrock:InvokeModel permission
export AWS_DEFAULT_REGION=us-east-1
claude

# Option C: Profile
export AWS_PROFILE=my-profile
claude

# Configure model ARN for Bedrock
export ANTHROPIC_MODEL="us.anthropic.claude-sonnet-4-6-20251001-v1:0"
```

### Google Vertex AI

```bash
export CLAUDE_CODE_USE_VERTEX=1
export CLOUD_ML_REGION=us-east5
export ANTHROPIC_VERTEX_PROJECT_ID=my-gcp-project

# Authenticate with gcloud
gcloud auth application-default login
claude
```

### Azure Foundry

```bash
export CLAUDE_CODE_USE_FOUNDRY=1
export ANTHROPIC_FOUNDRY_ENDPOINT="https://my-endpoint.azure.com"
export AZURE_API_KEY="my-azure-key"
claude
```

---

## Auth for Team/Enterprise

```bash
# Claude for Teams — login with team workspace
claude auth login
# Prompts: personal or team account?
# Select team → authenticates with team plan

# Claude Console (API billing, SSO)
claude auth login --console
# Uses Console credentials
# Can configure SSO: Console → Settings → SSO

# SSO login
claude auth login --sso
# Redirects to configured SSO provider (Okta, Google, etc.)
```

---

## Credential Storage Details

```
  macOS:
    Security framework Keychain
    Entry: "Claude Code — user@example.com"
    Access: claude binary only (sandboxed)
    View: Keychain Access app

  Linux:
    ~/.claude/.credentials.json
    Permissions: 0600 (owner read/write only)
    Format: JSON with access_token, refresh_token, expiry

  Windows:
    Windows Credential Manager
    OR ~/.claude/.credentials.json (fallback)

  Custom location:
    export CLAUDE_CONFIG_DIR=/path/to/config
    Credentials stored in $CLAUDE_CONFIG_DIR/.credentials.json
```

---

## Token Refresh

```
  Access token expires → Claude Code detects 401 response
          │
          ▼
  Attempt token refresh using refresh_token
          │
          ▼
  POST https://auth.anthropic.com/token
  { grant_type: "refresh_token", refresh_token: <token> }
          │
          ▼
  New access_token stored
  Request retried automatically
          │
          ▼
  If refresh_token also expired:
          │
          ▼
  User prompted to run: claude auth login
```

---

## Auth Status and Troubleshooting

```bash
# Check current auth status
claude auth status

# Output:
# ✓ Logged in as: user@example.com
# Plan: Pro
# Auth type: OAuth
# Token expires: 2026-04-15

# Text format for scripting
claude auth status --text
# claude-code authenticated: user@example.com (Pro)

# Diagnose auth issues
/doctor
# Checks connectivity, token validity, API access

# Log out (clears all stored credentials)
claude auth logout

# Login again
claude auth login
```

---

## LLM Gateway Authentication

For companies routing through their own gateway:

```bash
# Bearer token to your gateway
export ANTHROPIC_AUTH_TOKEN="Bearer my-gateway-token"

# Custom base URL
export ANTHROPIC_BASE_URL="https://gateway.company.com/anthropic"

# Both together
export ANTHROPIC_AUTH_TOKEN="Bearer token"
export ANTHROPIC_BASE_URL="https://gateway.company.com"
claude
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  AUTHENTICATION CHEATSHEET                                     │
├────────────────────────────────────────────────────────────────┤
│  Check status   : claude auth status                           │
│  Login          : claude auth login                            │
│  Logout         : claude auth logout                           │
│  API key        : export ANTHROPIC_API_KEY=sk-ant-...         │
│  Bearer token   : export ANTHROPIC_AUTH_TOKEN=Bearer...       │
│  Dynamic creds  : apiKeyHelper in settings.json               │
│  AWS Bedrock    : CLAUDE_CODE_USE_BEDROCK=1 + aws configure   │
│  GCP Vertex     : CLAUDE_CODE_USE_VERTEX=1 + gcloud auth      │
│  Custom gateway : ANTHROPIC_BASE_URL=https://...              │
│  Credential loc : macOS Keychain / ~/.claude/.credentials.json│
│  Token refresh  : automatic on 401, prompts on expiry         │
└────────────────────────────────────────────────────────────────┘
```
