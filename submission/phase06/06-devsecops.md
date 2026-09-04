# Phase 6 — DevSecOps: Security Gates, Vulnerability Fixes & Secrets

## Objective

Run the security toolchain, triage the findings, fix planted vulnerabilities at the appropriate layer, add or use security gates to prevent regression, and verify that application credentials are loaded from OpenBao rather than environment variables.

## 1. Enable the planted vulnerabilities

Enabled the deliberately insecure mode:

```bash
make insecure-on
```

This enabled `INSECURE_MODE` for the orders service and exposed the planted `/insecure/*` endpoints.

Evidence:
- `45-Live Planted 6 vulnerabilities.png`

The exercise contains six planted vulnerability classes:
1. SQL injection
2. IDOR
3. Error disclosure
4. Weak password hashing
5. Missing rate limiting
6. SSRF

The exercise demonstrates that automated security tooling does not necessarily detect every vulnerability.

## 2. SAST scan

Ran:

```bash
./security/scan-all.sh sast
```

The initial SAST scan reported 18 blocking findings.

Evidence:
- `46-sast-findings.png`

After fixing SQL injection, error disclosure, and weak password hashing, the blocking findings decreased from 18 to 15.

The findings were triaged instead of simply suppressed.

## 3. Security findings triage

| Finding | Classification | Action |
|---|---|---|
| SQL injection | Teaching vulnerability | Fixed using a parameterized SQL query |
| IDOR | Teaching vulnerability | Identified; not fixed during this exercise |
| Error/stack-trace disclosure | Teaching vulnerability | Fixed with a generic error response and reference ID |
| Weak password hashing | Teaching vulnerability | Fixed by replacing MD5 with scrypt |
| Missing rate limiting | Teaching vulnerability | Identified; not fixed during this exercise |
| SSRF | Teaching vulnerability | Identified; not intentionally fixed during this exercise |
| Dependabot configuration findings | Real hardening finding | Requires dependency-update configuration improvement |
| Infracost workflow secret exposure | Real CI/CD security finding | Requires workflow hardening |
| Public IP on AWS subnet | IaC/security finding | Requires review according to intended architecture |
| GCP Cloud SQL insecure SSL configuration | Real security finding | TLS should be enforced |
| Nginx Host-header handling | Web security finding | Expected hosts should be validated |
| Vulnerable `qs` dependency | SCA finding | Dependency update/remediation required |
| Docker `:latest` image usage | Supply-chain finding | Images should be pinned to immutable versions/digests |
| Web container running as root | Container security finding | Use a non-root runtime user |
| Unrestricted AWS egress | IaC/security finding | Restrict outbound access where practical |
| Azure Key Vault network ACL | IaC/security finding | Harden network access |
| Missing healthchecks | Hardening warning | Add appropriate application/container healthchecks |

The full security scan was intentionally not made completely green because Daig contains deliberate vulnerabilities and additional findings requiring separate remediation.

## 4. SQL injection — VULN-1

### Before

The original vulnerable search endpoint constructed SQL using user-controlled input:

```javascript
const sql = `SELECT id, name, area FROM restaurants WHERE name LIKE '%${term}%'`;
```

A quote character demonstrated the problem and produced a PostgreSQL syntax error.

Evidence:
- `47-sqli-before-fix.png`

### Fix

Changed the query to use a parameterized PostgreSQL query:

```javascript
const {rows}=await q(
  'search',
  'SELECT id,name,area FROM restaurants WHERE name ILIKE $1',
  [`%${term}%`]
);
```

The user-controlled value is supplied separately as a query parameter and is therefore treated as data rather than SQL syntax.

### After

The same test after rebuilding the orders service returned:

```json
{
  "results": []
}
```

Evidence:
- `48-sqli-after-fix.png`

### Security gate

The repository already contains a custom Semgrep SQL-concatenation rule in:

```text
security/semgrep/daig.yml
```

The rule is integrated into the SAST process through:

```text
security/scan-all.sh
```

After the fix, the corresponding SQL-concatenation finding was no longer present in:

```text
security/reports/semgrep.json
```

This provides a regression gate for this vulnerability class.

## 5. Error disclosure — VULN-3

The planted error endpoint exposed internal information such as database errors, stack traces, database connection information, and Node.js runtime information.

The endpoint was changed so detailed information is logged internally while the client receives only a generic error response and a UUID reference.

Example:

```json
{
  "error": "internal error",
  "reference": "..."
}
```

Evidence:
- `49-error-disclosure-after-fix.png`

The SAST findings decreased after this change.

## 6. Weak password hashing — VULN-4

The planted registration endpoint used MD5 for password hashing.

The implementation was changed to use Node.js `crypto.scryptSync()` with a randomly generated salt.

Example response:

```json
{
  "username": "...",
  "password_hash": "...",
  "salt": "...",
  "algorithm": "scrypt"
}
```

Evidence:
- `50-weak-hash-after-fix.png`

The weak-hashing SAST finding disappeared after the change.

## 7. Security verification

Ran:

```bash
./chaos/day6-security.sh verify
```

The verification reported:

```text
VULN-1 SQL injection                   fixed or unreachable
VULN-3 error disclosure                fixed or unreachable
VULN-4 weak hashing                    fixed or unreachable
VULN-6 SSRF                            fixed or unreachable
```

Evidence:
- `51-security-verification.png`

VULN-1, VULN-3, and VULN-4 were intentionally remediated during this exercise.

VULN-6 is not claimed as fixed because no intentional VULN-6 remediation was performed.

The verification script also explains that VULN-2 (IDOR) and VULN-5 (missing rate limiting) cannot be reliably probed by this automated check.

## 8. OpenBao secrets management

Started OpenBao with:

```bash
make vault-up
```

The application services were configured to receive the OpenBao connection and AppRole values:

```text
BAO_ADDR
BAO_ROLE_ID
BAO_SECRET_ID
```

The orders service was then recreated using:

```bash
make vault-app
```

The application secret-resolution code uses these values to authenticate to OpenBao.

## 9. Orders least-privilege policy

The orders policy is defined in:

```text
vault/policies/orders.hcl
```

Orders can read its required database and application secrets:

```hcl
path "daig/data/database" {
  capabilities = ["read"]
}

path "daig/data/app" {
  capabilities = ["read"]
}
```

It can obtain its own dynamic database credentials:

```hcl
path "daig-db/creds/orders" {
  capabilities = ["read"]
}
```

It can renew its own leases and token:

```hcl
path "sys/leases/renew" {
  capabilities = ["update"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}
```

Access to the payment secret is explicitly denied:

```hcl
path "daig/data/payment" {
  capabilities = ["deny"]
}
```

System paths are also denied:

```hcl
path "sys/*" {
  capabilities = ["deny"]
}
```

This implements least privilege.

## 10. OpenBao credential-source verification

Ran:

```bash
make vault-app
```

The application reported:

```text
"credential_source":"openbao"
```

This proves that the application resolved its credentials through OpenBao rather than using the environment fallback.

Evidence:
- `52-credential-source-openbao.png`

## 11. OpenBao payment-secret denial

The orders AppRole was used to attempt to read:

```text
daig/data/payment
```

OpenBao rejected the request:

```text
Code: 403. Errors:

* permission denied
```

Evidence:
- `53-openbao-payment-denied.png`

This proves that the orders AppRole cannot access the payment secret.

## 12. Why the security gates are ordered this way

A sensible security-gate order is:

```text
Secrets
   ↓
SAST
   ↓
SCA
   ↓
IaC / container policy
   ↓
Build
   ↓
DAST
```

Secrets checks happen early because leaked credentials can result in immediate compromise.

SAST checks source-code security problems before deployment.

SCA checks third-party dependencies for known vulnerabilities.

IaC and container policy checks infrastructure and image configuration.

The build should proceed only after earlier security checks pass.

DAST runs against a live application and is later because it requires a running environment and is generally more expensive.

## 13. Why `secrets.js` exits with status 78

When OpenBao is configured, the application must fail closed if OpenBao is unreachable or unauthorized.

It must not silently switch to environment variables.

The intended behavior is:

```text
OpenBao configured
       ↓
OpenBao unavailable/unauthorized
       ↓
DO NOT use environment fallback
       ↓
Fail closed
```

Exit status `78` indicates a configuration problem.

Failing closed prevents an operator from believing centrally managed secrets are being used when the application has actually fallen back to potentially unsafe environment credentials.

## 14. Security gate lesson

Security scanning is not equivalent to security.

The SAST scanner detected several application vulnerabilities, but it did not detect every planted vulnerability.

IDOR and missing rate limiting require application-specific testing and expected-behavior knowledge.

Therefore:

```text
Scanner findings
       +
Manual triage
       +
Runtime verification
       +
Regression gates
       =
Stronger security process
```

A scanner is one control within a broader defense-in-depth process.

## 15. Evidence summary

| Evidence | Purpose |
|---|---|
| `45-Live Planted 6 vulnerabilities.png` | Shows planted vulnerabilities enabled |
| `46-sast-findings.png` | Initial SAST findings |
| `47-sqli-before-fix.png` | SQL injection before remediation |
| `48-sqli-after-fix.png` | SQL injection after remediation |
| `49-error-disclosure-after-fix.png` | Error disclosure remediation |
| `50-weak-hash-after-fix.png` | Weak hashing remediation |
| `51-security-verification.png` | Automated vulnerability verification |
| `52-credential-source-openbao.png` | Application loading credentials from OpenBao |
| `53-openbao-payment-denied.png` | Orders AppRole denied payment-secret access |

## 16. Phase 6 result

Phase 6 demonstrated a complete DevSecOps security workflow:

1. Enabled deliberately planted vulnerabilities.
2. Ran SAST and the full security toolchain.
3. Triaged scanner findings into teaching vulnerabilities and real hardening findings.
4. Fixed VULN-1 SQL injection using parameterized SQL.
5. Fixed VULN-3 error disclosure using a generic client response and internal reference.
6. Fixed VULN-4 weak hashing by replacing MD5 with scrypt and a random salt.
7. Verified the remediated vulnerabilities.
8. Used the existing Semgrep SQL-concatenation rule as a regression gate.
9. Configured OpenBao AppRole credentials for the application.
10. Verified `"credential_source":"openbao"`.
11. Verified that the orders AppRole receives HTTP 403 when attempting to read the payment secret.
12. Documented fail-closed secret handling and ordered security gates.

The remaining scanner findings were triaged rather than blindly suppressed.
