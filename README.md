# Cedarling-OPA Terraform JWT Authorization Demo

A working GitHub Actions demo that gates Terraform operations using
**cryptographically verified GitHub OIDC JWTs** and **Cedar policies** evaluated
by [Cedarling](https://github.com/JanssenProject/jans/tree/main/jans-cedarling)
running inside OPA.

No secrets, no service-account credentials, no external authorization server —
the OPA container spins up inside each CI job, validates the GitHub-signed JWT
against GitHub's JWKS endpoint, and allows or denies the Terraform command based
on the repository name, Git ref, and GitHub Environment approval status.

---

## Quick setup

1. **Fork or clone this repository** into your GitHub account.

2. **Set your repository name in the Cedar policies** — open each of the three
   files in `policy-store/policies/` and replace `moabu/cedarling-opa-terraform-jwt` with your
   repository's full name (e.g. `acme/infra`):

   ```bash
   # macOS
   sed -i '' 's|moabu/cedarling-opa-terraform-jwt|your-org/your-repo|g' policy-store/policies/*.cedar
   # Linux
   sed -i 's|moabu/cedarling-opa-terraform-jwt|your-org/your-repo|g' policy-store/policies/*.cedar
   ```

3. **Push to `main`** — the workflow triggers automatically.

4. **Create the `production` GitHub Environment** with required reviewers so
   production applies are gated behind human approval:
   - Go to **Settings → Environments → New environment** → name it `production`.
   - Under **Deployment protection rules**, enable **Required reviewers** and add
     at least one reviewer.
   - Click **Save protection rules**.

5. **Open a pull request** to trigger the plan job, then merge to `main` to
   trigger the staging apply. The production apply will pause at the Environment
   gate until a reviewer approves.

---

## Authorization model

| Scenario | Plan | Apply | Destroy |
|---|:---:|:---:|:---:|
| Any branch, trusted repo | ✓ | ✗ | ✗ |
| `main` branch, trusted repo, non-prod workspace | ✓ | ✓ | ✗ |
| `main` branch, trusted repo, `production` Environment approved | ✓ | ✓ | ✗ |
| `Destroy` from any CI pipeline | ✗ | ✗ | ✗ |

`Destroy` is implicitly denied for all CI pipelines — no Cedar policy allows it.

---

## How it works

```
GitHub Actions runner
  │  (permissions: id-token: write)
  │
  ├─► docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
  │       OPA + Cedarling starts at http://localhost:8181
  │       Fetches GitHub's JWKS at startup for signature validation
  │
  ├─► composite action: tf-jwt-authz (from JanssenProject/jans)
  │       │
  │       ├─ Fetches GitHub OIDC token (ACTIONS_ID_TOKEN_REQUEST_URL)
  │       │
  │       └─ POST /v1/data/infra/terraform_jwt
  │            { tokens: [{ mapping: "CI::GitHubWorkflow", payload: "<jwt>" }],
  │              action: "CI::Action::\"Plan\"",
  │              resource: { entity_type: "CI::TerraformWorkspace", id: "staging" } }
  │                │
  │                ├─ Validate JWT signature (GitHub JWKS)
  │                ├─ Map claims → CI::GitHubWorkflow entity
  │                │    repository: "your-org/your-repo"
  │                │    ref:        "refs/heads/main"
  │                │    environment: "production"  ← only after Environment approval
  │                └─ Evaluate Cedar policies → ALLOW / DENY
  │
  └─► terraform plan / apply / (deny destroy)
```

The composite action (`tf-jwt-authz`) is consumed directly from
[JanssenProject/jans](https://github.com/JanssenProject/jans/tree/main/jans-cedarling/cedarling_opa/demo/terraform-jwt/.github/actions/tf-jwt-authz)
— no local copy is needed.

### Production gate

The `ci-permit-apply-prod-via-environment` Cedar policy requires the OIDC token
to contain `environment == "production"`. GitHub **only** injects this claim
when:

1. The workflow job declares `environment: production`.
2. A designated reviewer has explicitly approved the deployment.

Without approval the job never starts — the token is never issued. The Cedar
policy cannot be bypassed by adding `environment: production` to a workflow file
alone.

---

## Running locally (demo mode)

Demo mode disables JWT signature validation so you can test with crafted tokens
— no GitHub Actions credentials needed.

```bash
docker compose up
```

OPA starts at `http://localhost:8181`. Generate a test JWT:

```bash
# pip install pyjwt
python3 - <<'EOF'
import jwt, time
claims = {
    "iss": "https://token.actions.githubusercontent.com",
    "sub": "repo:your-org/your-repo:ref:refs/heads/main",
    "jti": "demo-plan-001",
    "repository": "your-org/your-repo",
    "ref": "refs/heads/main",
    "workflow": ".github/workflows/terraform.yml",
    "environment": "",
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600,
    "aud": "cedarling-terraform",
}
print(jwt.encode(claims, "demo-secret", algorithm="HS256"))
EOF
```

Query OPA directly:

```bash
export TF_JWT="<paste token>"

curl -s -X POST http://localhost:8181/v1/data/infra/terraform_jwt \
  -H "Content-Type: application/json" \
  -d "{
    \"input\": {
      \"tokens\": [{ \"mapping\": \"CI::GitHubWorkflow\", \"payload\": \"${TF_JWT}\" }],
      \"action\": \"CI::Action::\\\"Plan\\\"\",
      \"resource\": {
        \"cedar_entity_mapping\": { \"entity_type\": \"CI::TerraformWorkspace\", \"id\": \"staging\" }
      },
      \"context\": { \"current_time\": $(date +%s) }
    }
  }" | jq .
```

Stop the server:

```bash
docker compose down
```

---

## Repository structure

```
.
├── .github/
│   └── workflows/
│       └── terraform.yml          Live demo workflow (plan / apply-staging / apply-production)
├── policy-store/
│   ├── metadata.json
│   ├── schema.cedarschema         Cedar entity schema (CI namespace)
│   ├── trusted-issuers/
│   │   └── github-actions.json   GitHub Actions OIDC issuer config
│   └── policies/
│       ├── ci-permit-plan.cedar
│       ├── ci-permit-apply-main-non-prod.cedar
│       └── ci-permit-apply-prod-via-environment.cedar
├── rego/
│   └── terraform_jwt.rego         OPA policy (delegates to Cedarling)
├── docker-compose.yml             Quick start — demo mode (sig validation OFF)
├── docker-compose.prod.yml        Production overlay (sig validation ON)
├── opa-config.json                Production Cedarling config
├── opa-config-demo.json           Demo Cedarling config
└── main.tf                        Minimal Terraform config (hashicorp/local, no credentials)
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `DENIED` even with correct repo name | `moabu/cedarling-opa-terraform-jwt` still in policy files | Replace with your actual repo name and push |
| `DENIED` for production apply | `environment` claim missing | Ensure the job uses `environment: production` and has been approved |
| OPA health-check times out | Image pull slow or JWKS fetch blocked | Check runner has outbound HTTPS to `token.actions.githubusercontent.com` |
| `id-token: write` error | Permission missing | Ensure each job declares `permissions: id-token: write` |

---

## Further reading

- [Full demo documentation](https://github.com/JanssenProject/jans/tree/main/jans-cedarling/cedarling_opa/demo/terraform-jwt) — architecture deep-dive, decision matrix, and local build instructions
- [Cedarling](https://github.com/JanssenProject/jans/tree/main/jans-cedarling) — the authorization engine powering this demo
- [Cedar policy language](https://www.cedarpolicy.com/)
