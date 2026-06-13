# Exercise 7.2 — Multi-Env Layout and GitHub Environment Promotion

**Course:** Optimizaciones y Desempeño — Cloud Deployment Automation
**Session:** 7 — June 4, 2026

---

## Teacher's Intent

This exercise targets a gap that trips up many teams moving from a working-but-opaque CI pipeline to one that is production-grade: *granularity of feedback*. When `terraform fmt`, `validate`, and `plan` all live inside one job, GitHub reports a single status check. A reviewer looking at a pull request cannot tell whether the failure was a formatting nitpick or a broken resource configuration. Separating them into three named jobs makes each check independently visible, independently re-runnable, and independently dismissible — exactly how mature infrastructure teams operate.

The second gap this exercise closes is **artifact-based promotion**. Students often re-run `terraform plan` inside `apply`. This is dangerous: the plan GitHub showed the reviewer is not the plan that gets applied — a new plan runs against the live state at apply time and the diff may have drifted. Uploading the plan binary and downloading it in the apply job guarantees that what was reviewed is exactly what is applied.

The third gap is **environment protection**. GitHub Environments are the native mechanism for gating promotions. Declaring `environment: staging` on the `apply-staging` job causes GitHub to pause execution and require a named reviewer to approve before any state mutation happens in staging. Students learn to see environment gates not as bureaucracy but as a forcing function that creates an audit trail (who approved, when) at no extra tooling cost.

The sequence of tasks is intentional: create the environments in GitHub before rewriting the workflow, so students appreciate that the YAML `environment:` key is *declaring intent* against a gate that must already exist — not creating it.

---

## Repository Structure

```
.github/
└── workflows/
    └── terraform-cd.yml     ← rewritten across tasks 2–3
infra/
├── provider.tf
├── main.tf
├── variables.tf
└── envs/
    ├── dev/
    │   ├── dev.tfvars
    │   └── backend-dev.hcl
    └── staging/
        ├── staging.tfvars
        └── backend-staging.hcl
evidence/
└── pr-url.txt
```

---

## Step-by-Step Implementation

### Step 1 — Scaffold

Copy all provided Terraform files into the repo exactly as given. The starter `terraform-cd.yml` is included verbatim — it is the baseline students are asked to improve. Replace `YOUR_BUCKET_NAME` in both backend HCL files with your actual S3 bucket name.

**Teaching point:** The repo structure separates environment config (`envs/dev/`, `envs/staging/`) from Terraform logic (`provider.tf`, `main.tf`, `variables.tf`). This layout makes it possible to run the same Terraform code against multiple backends without duplicating resource definitions.

---

### Step 2 — Task 1: Create GitHub Environments

GitHub Environments are created through the repository UI, not through code. This step produces no file changes — the evidence is a screenshot of the configured environments.

Navigate to **Settings → Environments** and create:
- `dev` — no protection rules
- `staging` — enable **Required reviewers**, add yourself

**Teaching point:** The `environment:` key in a workflow job is a *reference* — GitHub looks up the named environment and enforces whatever protection rules are attached. If the environment doesn't exist yet, the job runs without any gate. Creating the environment first, before the workflow references it, teaches students to treat the gate as infrastructure that is provisioned separately from the pipeline that uses it.

**Evidence:**
<!-- Screenshot of GitHub → Settings → Environments showing both dev and staging configured -->
_[Evidence: screenshot of both environments in GitHub — to be added after Task 1]_

---

### Step 3 — Task 2: Split validation into three named PR jobs

Rewrite `terraform-cd.yml` to replace the single `terraform-ci` job with three independent jobs:

| Job | Steps | Credentials needed? |
|-----|-------|---------------------|
| `terraform-fmt` | checkout, setup-terraform, `fmt -check` | No |
| `terraform-validate` | checkout, setup-terraform, `init -backend=false`, `validate` | No |
| `terraform-plan` | checkout, setup-terraform, `init` (full), `plan -out=tfplan`, `show > plan.txt`, upload artifact, post PR comment | Yes |

**Teaching point:** `terraform init -backend=false` lets validate run without AWS credentials. The backend config is only needed when Terraform must reach S3 to lock and read state. Teaching students to avoid credentials in jobs that don't need them is a least-privilege habit that reduces the blast radius of a compromised runner.

The PR comment step uses `actions/github-script@v7` to read `plan.txt` and post it as a comment. This closes the feedback loop: the reviewer sees the plan diff directly in the PR without leaving GitHub.

**Evidence:**
<!-- Screenshot or link showing three separate status checks on a PR -->
_[Evidence: PR showing terraform-fmt, terraform-validate, terraform-plan as individual checks — to be added after Task 4]_

---

### Step 4 — Task 3: Add apply jobs

Add two jobs that are gated on `push: [main]` (not on pull_request):

- `apply-dev`: needs `[terraform-fmt, terraform-validate, terraform-plan]`; declares `environment: dev`; downloads the `tfplan-dev` artifact; runs `terraform apply tfplan`
- `apply-staging`: needs `[apply-dev]`; declares `environment: staging`; downloads the same artifact; runs `terraform apply -var-file=envs/staging/staging.tfvars`

**Teaching point:** `apply-staging` does not re-plan — it applies the same binary artifact that was uploaded during the PR checks. The staging apply uses the staging tfvars but the *dev plan artifact*. In a real system each environment would have its own plan; here the point is that apply never re-plans — it always consumes a reviewed artifact. The `needs:` chain `apply-dev → apply-staging` creates a sequential promotion gate even without environment rules.

**Evidence:**
<!-- Link to the run showing apply-dev succeeded and apply-staging waited for approval -->
_[Evidence: link to Actions run showing the promotion gate — to be added after Task 4]_

---

### Step 5 — Task 4: Open a PR and verify

Create a feature branch, add a comment to `main.tf`, push, and open a PR targeting `main`.

**Verification checklist:**
- [ ] Three individual status checks appear on the PR
- [ ] `terraform-plan` posts a plan comment on the PR
- [ ] Merging triggers `apply-dev` automatically
- [ ] `apply-staging` pauses for manual approval
- [ ] Both apply jobs succeed after approval

**Evidence:**
_[Evidence: pr-url.txt populated — to be added after Task 4]_

---

## Evidence

### Task 1 — GitHub Environments

_[To be populated after Task 1]_

### Task 2 — Three PR Status Checks

_[To be populated after Task 4 — screenshot or PR link showing three checks]_

### Task 3 — Apply Jobs and Promotion Gate

_[To be populated after Task 4 — Actions run link]_

### Task 4 — PR URL

_[To be populated — see evidence/pr-url.txt]_
