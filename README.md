# GitHub Reconnaissance & Public-Exposure Assessment
## Target: MATLAB Online (`matlab.mathworks.com`) — Bugcrowd "MATLAB Online"

**Assessment type:** Passive OSINT / public-source reconnaissance (GitHub)
**Date:** 2026-08-21
**Method:** Public GitHub web + authenticated code search + public documentation. **No traffic was sent to the live target and no credentials were used or tested.**
**Program:** `bugcrowd.com/engagements/matlab-online` (confirmed to exist; full brief is login-gated)

---

## ⚠️ Scope caveat (read first)

The official Bugcrowd brief is behind authentication and could not be retrieved, so the exact in-scope / out-of-scope asset list was **not** confirmed. This report treats **`matlab.mathworks.com`** as the sole confirmed in-scope asset (per the engagement name) and marks everything else — other `*.mathworks.com` hosts and all GitHub repositories — as **reconnaissance sources / out-of-scope for testing** unless you confirm otherwise from the authenticated brief. **Do not test any host below until you have verified it against the program scope.**

---

## 1. Executive Summary

- MathWorks maintains a large public GitHub footprint: **~547 repos** in the primary `mathworks` org and **~63** in the `mathworks-ref-arch` (cloud reference-architecture) org, plus product-focused orgs (`matlab-deep-learning`, etc.). The overwhelming majority are MATLAB/Simulink **educational examples and toolboxes** with no security relevance to the web target.
- **No valid exposed secrets were found.** Every hit for secret indicators in MathWorks-owned repos resolved to a **false positive** (SDK docstring, `YOUR_...` placeholder, or an upstream Linux-kernel test key). This is the expected, reassuring result for curated example repos.
- **No third-party repository was observed leaking the target's credentials** (session cookies, passwords, API keys for `matlab.mathworks.com`). The many domain matches are the legitimate **"Open in MATLAB Online" badge**.
- The most useful outputs are **architectural leads**, not leaks:
  - A real in-scope endpoint — **`matlab.mathworks.com/open/github/v1?repo=…&file=…`** — the "Open in MATLAB Online" deep-link, which accepts repo/file parameters and is worth manual testing (open-redirect / host-validation / parameter handling).
  - The **open-source `matlab-proxy`** component reveals the browser-serving auth model (token-based; **token accepted via URL parameter**), a useful white-box map for analogous behaviors to probe on MATLAB Online.
  - A historical **internal hostname disclosure** — `www-internal.mathworks.com` — in old MATLAB doc tooling copied across many public forks (informational; almost certainly not externally reachable and out of scope).
- No published security advisories/CVEs were found for `matlab.mathworks.com`, `matlab-proxy`, or `MATLAB-language-server`.

**Bottom line:** Public GitHub exposure is low. There are no credential leaks to report, but there are legitimate **attack-surface leads on the in-scope web app** that require authenticated, manual testing under the program's rules (Section 9).

---

## 2. Public Repository Inventory (security-relevant subset)

Only repos relevant to the web target / infrastructure are listed. The other ~590 repos are MATLAB/Simulink teaching examples.

### Primary org: `github.com/mathworks`

| Repository | Lang | Relevance | Notes |
|---|---|---|---|
| **matlab-proxy** | Python | **High** | Serves a MATLAB desktop in a browser tab. Closest public analog to MATLAB Online's browser delivery. Token-based auth (`MWI_ENABLE_TOKEN_AUTH`, `MWI_AUTH_TOKEN`), optional SSL. Has `.github/workflows`, `SECURITY.md`. |
| **MATLAB-language-server** | TypeScript | Medium | Language server behind the VS Code extension and editor tooling; input/document handling worth review. |
| **MATLAB-extension-for-vscode** | TypeScript | Medium | Editor integration; talks to the language server. |
| **parallel-server-proxy** | Go | Medium | "Proxy server for MATLAB client traffic to Parallel Server clusters." |
| **matlab-azure-devops-extension** | TypeScript | Low/Med | CI integration; indicates Azure DevOps usage. |
| **data-explorer-vscode** | TypeScript | Low | Simulink Data Explorer VS Code extension. |
| matlab-parallel-awsbatch-plugin | MATLAB | Low | AWS Batch plugin (contained an `AWS_SECRET_ACCESS_KEY` **placeholder** — see §4). |

### Cloud reference-architecture org: `github.com/mathworks-ref-arch` (~63 repos)

These are **customer-deployable** IaC templates (you run them in *your* cloud). They are **not** MathWorks' own production infra for `matlab.mathworks.com`, but they reveal the technology conventions.

| Repository | Lang / IaC | Notes |
|---|---|---|
| matlab-on-aws | HCL / CloudFormation | MATLAB desktop on AWS w/ Remote Desktop. |
| matlab-on-azure | HCL (Terraform) | MATLAB desktop via Azure Resource Manager. |
| matlab-production-server-on-aws | Python / CloudFormation | Production Server on AWS. |
| matlab-production-server-on-kubernetes | Go Template (Helm) | Production Server on K8s. |
| matlab-parallel-server-on-{aws,azure,eks,kubernetes} | HCL/Python/Go | Parallel cluster deployments. |
| license-manager-for-matlab-on-{aws,azure} | Mixed | Network License Manager (MLM) automation. |
| matlab-web-app-server-on-azure | Python | Web App Server on Azure. |
| container-images | Dockerfile | Official MathWorks images on Docker Hub. |
| matlab-dockerfile | Python/Dockerfile | Build a container with MATLAB installed. |
| matlab-codespaces | Dockerfile | Dev-container config for GitHub Codespaces. |

Other MathWorks orgs observed in results: **`matlab-deep-learning`** (model repos, heavy "Open in MATLAB Online" badge usage), plus robotics/other product orgs.

---

## 3. Technology Stack (as referenced in public code)

- **Languages:** MATLAB (dominant), Python (3.10–3.14 for matlab-proxy), TypeScript / JavaScript / Node.js (24+), Go, C/C++, Shell/PowerShell, Jupyter.
- **Frontend:** Node.js build + React GUI (matlab-proxy `gui/`).
- **Infrastructure-as-Code:** Terraform (HCL), CloudFormation, Azure Resource Manager, Helm, Dockerfiles, Kubernetes manifests.
- **Cloud:** AWS (EC2, EKS, CloudFormation), Azure (ARM, AKS, DevOps), Databricks, GitHub Codespaces, Docker Hub.
- **CI/CD:** GitHub Actions (`.github/workflows/` in matlab-proxy et al.), Azure DevOps (dedicated extension published).
- **Auth / identity:** MathWorks Account (online licensing / SSO), Network License Manager (MLM), token-based auth for browser proxy (`MWI_AUTH_TOKEN`).
- **Supporting:** RabbitMQ (parallel/production server), object storage (S3/Azure Blob via SDK wrappers), SSL/TLS configurable via env vars.

---

## 4. Potential Secret Exposure — **metadata only** (all false positives)

No complete secret values are reproduced. **No "Potentially Valid Secret" was found.**

| # | Repository | File | Indicator | Classification | Why |
|---|---|---|---|---|---|
| 1 | mathworks/arrow | `python/pyarrow/_azurefs.pyx` | `client_secret` | **Likely Documentation** | Docstring describing the Azure AD `ClientSecretCredential` **parameter**, not a value. (Repo is the Apache Arrow fork.) |
| 2 | mathworks/matlab-parallel-awsbatch-plugin | `README.md` | `AWS_SECRET_ACCESS_KEY` | **Likely Placeholder** | Literal `'YOUR_AWS_SECRET_ACCESS_KEY'` in a `setenv(...)` usage example. |
| 3 | mathworks/xilinx-linux | `tools/testing/selftests/sgx/sign_key.pem` | `BEGIN RSA PRIVATE KEY` | **Likely Test/Example** | Well-known **upstream Linux-kernel** SGX self-test key that ships in the kernel tree; repo is a kernel fork. Not a MathWorks secret. |

Cross-org search for `"matlab.mathworks.com"` combined with `cookie / password / token / authorization / secret / apikey`: **134 files, all noise** — matches were the "Open in MATLAB Online" badge plus ML "token/tokenizer" terminology. **No leaked target credentials observed.**

---

## 5. Domains / Subdomains / Hosts

| Host | Environment | Source | Scope status | Security relevance |
|---|---|---|---|---|
| **matlab.mathworks.com** | Production | Target / "Open in MATLAB Online" badge (2.4k files) | **In scope (assumed)** | The web target. Endpoint `/open/github/v1?repo=…&file=…` accepts external params — see §9. |
| api.mathworks.com | Production (public API) | `mobeets/mpm` → `api.mathworks.com/community/v1/search` | **Unknown — verify** | Public File Exchange / community API. Likely out of the named scope; confirm before testing. |
| www-internal.mathworks.com | **Internal** | `*/matlab-bgl` → `doc/mxdom2mbgl-html.xsl` (many forks) | **Out of scope** | Historical internal image-server hostname leaked in old MATLAB doc-gen XSL. Informational only; do **not** attempt to reach it. |
| login.mathworks.com | Production (SSO) | Known MathWorks Account host (inferred, not from a leak) | **Unknown — verify** | Account authentication / SSO that MATLAB Online relies on. |
| www.mathworks.com | Production (public) | Documentation links | Out of scope (public site) | Public marketing/docs. |

No private IPs, database hostnames, or non-public service DNS names were found in public code.

---

## 6. Attack-Surface Summary (publicly observable)

- **Web application:** `matlab.mathworks.com` (MATLAB Online) — the in-scope target.
- **Notable endpoint:** `/open/github/v1` deep-link (repo/file params) — highest-value in-scope lead.
- **Integrations to review:** GitHub and Google Drive connectors (OAuth flows), "open file from URL," the Live Editor, and file-sharing features.
- **Auth systems:** MathWorks Account SSO (`login.mathworks.com`), session cookies, connector/session tokens.
- **Related open-source (white-box aid, not targets):** `matlab-proxy` (note **token-via-URL** pattern → tokens can leak via `Referer`, logs, browser history), `MATLAB-language-server`, `parallel-server-proxy`.
- **Cloud/CI/CD:** AWS + Azure + Kubernetes reference architectures; GitHub Actions and Azure DevOps pipelines. These are customer-deployed templates, not the target's own production estate.

---

## 7. CI/CD Review (public workflows)

- `matlab-proxy` and peers use **GitHub Actions** (`.github/workflows/`). Spot review of the public workflow surface showed **no secrets echoed to logs, no secrets passed as CLI args, and no obviously unsafe `pull_request_target` + untrusted-input patterns** in the sampled files. A full audit of all 547 repos' workflows was **not** performed (out of proportion to the target).
- MathWorks publishes an **Azure DevOps** extension and a **GitHub Actions** setup action, confirming both CI systems are in use.

---

## 8. Findings Table

| Severity | Repository / Source | File / Location | Finding | Asset | Scope | Evidence | Impact | Recommended Action |
|---|---|---|---|---|---|---|---|---|
| **Informational** | mathworks-owned code search | `arrow`, `matlab-parallel-awsbatch-plugin`, `xilinx-linux` | Secret-indicator matches | GitHub repos | Recon only | Docstring / placeholder / kernel test key | None (false positives) | No action; documented for completeness |
| **Informational** | `*/matlab-bgl` forks | `doc/mxdom2mbgl-html.xsl` | Internal hostname `www-internal.mathworks.com` disclosed | Internal host | Out of scope | Hardcoded image URL in doc XSL | Minor infra info leak | Optional hygiene note to vendor; do not test |
| **Low (lead)** | Target site | `matlab.mathworks.com/open/github/v1` | External `repo`/`file` params in deep-link | `matlab.mathworks.com` | In scope (assumed) | 2.4k public badge URLs use it | Potential open-redirect / host-validation / param-injection | **Manual test under program rules** |
| **Informational** | `mathworks/matlab-proxy` | `Advanced-Usage.md`, `SECURITY.md` | Auth token may be passed via **URL parameter** | matlab-proxy (analog) | Recon only | Documented token delivery methods | Token leakage vector *if* mirrored on target | Check whether MATLAB Online leaks tokens via Referer/logs |

No **Critical/High/Medium** findings are asserted — none are supported by evidence from passive recon alone.

---

## 9. Manual Follow-Up (prioritized — requires authenticated testing YOU perform)

1. **Confirm the exact Bugcrowd scope first** (login-gated brief). Verify whether only `matlab.mathworks.com` is in scope, and the status of `api.mathworks.com` / other `*.mathworks.com`. Do nothing active until confirmed.
2. **`/open/github/v1` deep-link on the target** — highest-value in-scope test. Non-destructively check: does it validate the `repo` host (open redirect / arbitrary-repo clone)? How are `file`/other params handled? Any SSRF-style or reflected behavior? Stop at minimal proof.
3. **MATLAB Online auth/session review** — cookie flags (`HttpOnly`/`Secure`/`SameSite`), CSRF protection on state-changing actions, token-in-URL leakage via `Referer`, and the **GitHub/Google Drive OAuth** integration flows (redirect_uri validation, state param).
4. **Use `matlab-proxy` as a white-box map** — read its token/websocket/path-handling code, then test *only* `matlab.mathworks.com` for analogous issues.
5. **Internal hostname** `www-internal.mathworks.com` — report to the vendor as an informational infra-hygiene item only if the program accepts such reports; **do not attempt to resolve or reach it.**

---

## 10. Methodology & Limitations

- **Passive only.** All data came from public GitHub pages, authenticated GitHub **code search** (read-only), and public documentation. **No requests were sent to `matlab.mathworks.com`; no credentials, tokens, or keys were used, validated, or tested.** Section 12-style "safe validation" was intentionally deferred to you.
- **Scope brief unavailable** (Bugcrowd login-gated) — in/out-of-scope list not confirmed; `matlab.mathworks.com` assumed as the sole target.
- **Not exhaustive.** 547 + 63 repos were triaged by relevance, not read in full. Some regex code searches were truncated by GitHub ("query too expensive"). Secret searches cover the highest-signal indicators, not every permutation.
- **No secrets reproduced.** Any future real finding must be reported as metadata (repo/file/commit + why), never as a full value.

---

*Prepared as authorized bug-bounty reconnaissance. Findings distinguish confirmed facts from leads and false positives. Preserve exact repo/file/commit references for responsible disclosure.*
