# 58 · Terraform in Jenkins

> **Level:** ★★★ advanced
>
> **You will learn:** the same safe Terraform flow as topic 57, built with a **Jenkins declarative pipeline**: running Terraform in a pinned **Docker agent**, AWS credentials from an **EC2 instance role** (best) or the **AWS Credentials** binding, stages for checks → plan → **`input` approval** → apply of the **stashed** plan, `lock()` for one run per state, `disableConcurrentBuilds`, **multibranch** behaviour (plan on PRs, apply on `main`), and cleanup of secrets in the workspace.
>
> **You need:** topics 30 (credential chain), 36, 53, 56. Jenkins background: [Jenkins topics 23–24](../../Jenkins/notes/24-using-credentials-safely.md) (credentials), [31](../../Jenkins/notes/31-declarative-pipeline-anatomy.md) (declarative anatomy), [38](../../Jenkins/notes/38-when-input-milestone-lock.md) (`when`, `input`, `milestone`, `lock`), [40](../../Jenkins/notes/40-per-stage-and-docker-agents.md) (Docker agents), [43](../../Jenkins/notes/43-multibranch-pipelines.md) (multibranch), [61](../../Jenkins/notes/61-aws-and-k8s-primer.md) (AWS primer).
>
> **Part 9 · CI/CD** · Previous: [57 · Terraform in GitHub Actions](57-github-actions.md) · Next: [59 · Capstone part 1: foundation](59-capstone-1-foundation.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A `Jenkinsfile` in the infra repo runs Terraform in a **Docker container** with a pinned version, plans on every branch/PR, and on `main` pauses at an **`input`** step for approval before applying the **same saved plan**:

```groovy
pipeline {
  agent { docker { image 'hashicorp/terraform:1.17.2'; args '--entrypoint=' } }
  options { disableConcurrentBuilds(); timestamps() }
  stages {
    stage('Init & checks') { steps { sh 'terraform fmt -check -recursive && terraform init -input=false && terraform validate' } }
    stage('Plan')  { steps { sh 'terraform plan -out=tfplan -input=false'; stash name: 'plan', includes: 'tfplan' } }
    stage('Approve') { when { branch 'main' }; steps { input message: 'Apply this plan?', submitter: 'platform-leads' } }
    stage('Apply') { when { branch 'main' }; steps { unstash 'plan'; sh 'terraform apply -input=false tfplan' } }
  }
}
```

**Picture it.** The same building-contractor process as topic 56, but the foreman is Jenkins: he prepares the quote, waits at the door for the client's signature (`input`), then builds exactly what was signed.

**The one thing to know.** The `hashicorp/terraform` image's entrypoint is `terraform`, so Jenkins' Docker agent needs `args '--entrypoint='` to run `sh` steps. And never keep AWS keys in the Jenkinsfile: use the agent's **instance role** or a Jenkins credential.

**Try it now:** run the Jenkinsfile below in your Jenkins-in-Docker lab ([Jenkins topic 14](../../Jenkins/notes/14-lab-jenkins-in-docker.md)) against a local-provider project (no cloud needed), then switch it to AWS.

**You've got it if you can:** map each GitHub Actions concept from topic 57 to its Jenkins equivalent.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### GitHub Actions ↔ Jenkins mapping

| Concept | GitHub Actions (topic 57) | Jenkins |
|---|---|---|
| Pipeline definition | `.github/workflows/*.yml` | `Jenkinsfile` (declarative) |
| Pinned Terraform | `hashicorp/setup-terraform` with version | Docker agent `hashicorp/terraform:1.17.2`, or a custom agent image |
| AWS credentials | OIDC → role | **EC2 instance role** of the agent (best); or `withCredentials([aws(...)])` from the AWS Credentials plugin; or an assumed role via `credential_process`/STS |
| PR plans | `pull_request` trigger | **Multibranch** pipeline: PR/branch jobs (`CHANGE_ID` set) |
| Apply only on main | `if:` + push to main | `when { branch 'main' }` |
| Approval | `environment` with reviewers | `input` step with `submitter` |
| Saved plan between jobs | artifacts | `stash`/`unstash` (same build) or `archiveArtifacts` |
| One run per state | `concurrency` | `lock(resource: 'tf-dev-network')` / `disableConcurrentBuilds()` |
| Plan in the UI | Job summary / PR comment | Console log, `archiveArtifacts 'plan.txt'`, a PR comment via the GitHub plugin |

### Credentials options (best → worst)

| Option | How | Notes |
|---|---|---|
| **Instance role** on the agent (EC2 agents, or EKS pods with IRSA) | Nothing in the pipeline: the provider finds credentials via IMDS (topic 30, step 6) | No secrets in Jenkins at all; separate agent pools/roles for plan vs apply |
| **Assume a role per stage** | Instance role (or a base credential) + `assume_role` in the backend/provider, or `aws sts assume-role` in the stage | Different roles for plan and apply |
| **AWS Credentials binding** (`withCredentials([aws(credentialsId: 'aws-tf', ...)])`) | Jenkins stores an access key pair | Long-lived keys: rotate, scope tightly, restrict who can use the credential |

### Multibranch behaviour

With a **Multibranch Pipeline** ([Jenkins topic 43](../../Jenkins/notes/43-multibranch-pipelines.md)), every branch and PR gets its own job from the same `Jenkinsfile`:

- PR / feature branches → checks + **plan** only.
- `main` → plan + **approval** + **apply**.

Because PR builds run **untrusted code** with whatever credentials the agent has, PR plans should run on agents whose role is **read-only**, and apply on a separate, restricted agent label.

---

## Why it exists

Many companies run Jenkins, not GitHub Actions. The principles from topics 56–57 are tool-independent; this topic shows the Jenkins way to get the same guarantees: pinned tooling, no stored keys if possible, reviewed plan = applied plan, human approval for prod, one run per state.

---

## Analogy

The same **contractor change-order** process (topic 56). Jenkins is a foreman who:

- uses a sealed, labelled toolbox (pinned Docker image),
- wears the company's site badge rather than carrying master keys (instance role),
- writes the quote and pins it to the board (stash/archive the plan),
- waits at the gate until an authorised manager signs (`input` with `submitter`),
- lets only one crew work on a building at a time (`lock`).

---

## How it works inside

```
Multibranch scan / webhook ──▶ build for branch X (or PR-123)
  agent: docker run hashicorp/terraform:1.17.2 --entrypoint= (workspace mounted)
  stage Checks: fmt -check · init · validate · tflint (separate image/stage)
  stage Plan:   lock(tf-dev-network) { plan -out=tfplan; show -no-color > plan.txt }
                stash 'plan' (tfplan + lock file); archiveArtifacts plan.txt
  [main only]
  stage Approve: input (pauses the build; frees the executor if agent none at top level)
  stage Apply:   lock(tf-dev-network) { unstash 'plan'; init; apply tfplan }
  post: cleanWs()   (tfplan and .terraform contain secrets/providers)
```

Tip: don't hold an executor (or a `lock`) while waiting for a human. Put `input` in a stage with `agent none` so Jenkins doesn't block an agent for hours, and keep `lock` only around plan and apply.

---

## Code / commands

### The Jenkinsfile

`Jenkinsfile` at the repo root (root module in `envs/dev/network`):

```groovy
pipeline {
  agent none                                       // choose agents per stage

  options {
    disableConcurrentBuilds()                      // one build of this branch at a time
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timeout(time: 2, unit: 'HOURS')
  }

  environment {
    TF_IN_AUTOMATION = '1'
    TF_INPUT         = '0'
    ROOT             = 'envs/dev/network'
    AWS_REGION       = 'ap-south-1'
  }

  stages {
    stage('Checks & Plan') {
      agent {
        docker {
          image 'hashicorp/terraform:1.17.2'
          args  '--entrypoint='                    // the image's entrypoint is "terraform"
          label 'tf-plan'                          // agents whose instance role is READ-ONLY
        }
      }
      steps {
        dir(env.ROOT) {
          sh 'terraform fmt -check -recursive "$WORKSPACE"'
          sh 'terraform init -input=false -lockfile=readonly'
          sh 'terraform validate'
          lock(resource: "tf-${env.ROOT}") {
            sh '''
              set +e
              terraform plan -out=tfplan -lock-timeout=5m -detailed-exitcode -no-color > plan.txt
              code=$?
              cat plan.txt
              echo "$code" > plan.exitcode
              [ "$code" -eq 1 ] && exit 1 || exit 0
            '''
          }
          archiveArtifacts artifacts: 'plan.txt', fingerprint: true
          stash name: 'plan', includes: 'tfplan,.terraform.lock.hcl'
          script {
            env.PLAN_EXIT = readFile('plan.exitcode').trim()   // env vars set here persist to later stages
          }
        }
      }
      post {
        always { cleanWs() }
      }
    }

    stage('Approve') {
      when {
        allOf {
          branch 'main'
          expression { env.PLAN_EXIT == '2' }      // only if there are changes
        }
      }
      agent none
      steps {
        timeout(time: 1, unit: 'HOURS') {
          input message: "Apply the plan for ${env.ROOT}? (see plan.txt in the build artifacts)",
                ok: 'Apply',
                submitter: 'platform-leads'        // Jenkins user or group allowed to approve
        }
      }
    }

    stage('Apply') {
      when {
        allOf {
          branch 'main'
          expression { env.PLAN_EXIT == '2' }
        }
      }
      agent {
        docker {
          image 'hashicorp/terraform:1.17.2'
          args  '--entrypoint='
          label 'tf-apply'                         // separate agents whose role CAN write
        }
      }
      steps {
        dir(env.ROOT) {
          unstash 'plan'
          lock(resource: "tf-${env.ROOT}") {
            sh 'terraform init -input=false -lockfile=readonly'
            sh 'terraform apply -input=false -lock-timeout=5m tfplan'
          }
        }
      }
      post {
        always { cleanWs() }
      }
    }
  }
}
```

Key points:

- **Two agent labels**: `tf-plan` agents run with a read-only instance role; `tf-apply` agents (fewer, locked down) have the write role. Branch/PR builds never reach `tf-apply`, because those stages are guarded by `when { branch 'main' }`.
- ⚠️ **But the guard lives in the Jenkinsfile, and a PR can edit the Jenkinsfile.** A PR that deletes the `when` or asks for `label 'tf-apply'` in an early stage would run on the write agent. The `when` is a convenience, not a security boundary. Enforce the boundary outside the PR's control:
  - restrict the `tf-apply` agents/cloud template to the `main` branch job only (folder- or job-level node restrictions, e.g. the Job Restrictions plugin), or put apply in a **separate job** that PR builds can't trigger;
  - in the multibranch source, use **"Trust: Nobody"/"From users with Admin or Write permission"** so fork PRs build with the Jenkinsfile from the target branch, not from the PR;
  - give the apply role an OIDC/IAM trust that only `main` builds can satisfy, and require `CODEOWNERS` review on `Jenkinsfile`.
- **`lock(resource: ...)`** (Lockable Resources plugin) serialises plan/apply per state **across all jobs**, not just this branch.
- **`stash`/`unstash`** carry the exact `tfplan` to the apply stage in the same build.
- **`input` with `submitter`**: only that group can approve; `timeout` aborts if nobody does.
- **`cleanWs()`** removes `tfplan` and `.terraform/` (they contain secrets/large binaries).
- `-lockfile=readonly`, `-lock-timeout`, `-detailed-exitcode`: same reasons as topic 57.

- **`env.PLAN_EXIT`**: the plan's `-detailed-exitcode`, saved so the Approve/Apply stages run only when there are changes (`2`). Simpler pipelines skip this and always ask for approval on `main`.

### If you must use stored AWS keys

```groovy
withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                  credentialsId: 'aws-tf-plan',
                  accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
  sh 'terraform plan -out=tfplan -input=false'
}
```

The binding injects env vars (masked in logs) only inside the block ([Jenkins topic 24](../../Jenkins/notes/24-using-credentials-safely.md)). Prefer instance roles or short-lived assumed roles.

### Console output (excerpt)

```
[Pipeline] stage (Checks & Plan)
+ terraform init -input=false -lockfile=readonly
...
[Pipeline] lock
Trying to acquire lock on [Resource: tf-envs/dev/network]
Lock acquired on [Resource: tf-envs/dev/network]
+ terraform plan -out=tfplan -lock-timeout=5m -detailed-exitcode -no-color
...
Plan: 0 to add, 1 to change, 0 to destroy.
Lock released on resource [Resource: tf-envs/dev/network]
[Pipeline] stage (Approve)
Apply the plan for envs/dev/network? (see plan.txt in the build artifacts)
Apply or Abort
Approved by alice
[Pipeline] stage (Apply)
+ terraform apply -input=false -lock-timeout=5m tfplan
...
Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

---

## Hands-on lab

1. In your Jenkins-in-Docker lab, make sure the Docker agent works ([Jenkins topic 27](../../Jenkins/notes/27-lab-dood-setup.md)) and install the **Lockable Resources** plugin.
2. Create a Multibranch Pipeline for your infra repo. For a free first run, point `ROOT` at a **local-provider** root (e.g. topic 43's) and remove the agent labels.
3. Run a feature branch: checks + plan only. Run `main`: approve and apply.
4. Change to the AWS network root. On a Jenkins agent running on EC2, give the instance a read-only role and confirm `plan` works without any Jenkins credential; then set up a separate apply agent/role (or, in a simple lab, one role).
5. Start two builds that touch the same root and watch `lock` queue the second.
6. Abort an approval and confirm nothing was applied.

---

## Where you'll use it

- **Jenkins shops** running infrastructure pipelines next to application pipelines.
- **Self-hosted, network-restricted environments** where agents run inside private networks with instance roles.
- **Migration projects** from Jenkins to GitHub Actions (the mapping table above is your guide).

---

## When it breaks

| Error / symptom | Meaning | Fix |
|---|---|---|
| `terraform: Terraform has no command named "sh"` / container exits immediately | The image's entrypoint is `terraform`. | `args '--entrypoint='`. |
| `No valid credential sources found` | The agent has no instance role / no binding. | Attach a role to the agent instance; or `withCredentials`. |
| Build holds an executor for hours | `input` inside a stage with an agent. | `agent none` on the approval stage. |
| Two builds fight over the state lock | No cross-job serialisation. | `lock(resource: ...)` per state; `-lock-timeout`. |
| `Saved plan is stale` in Apply | Someone applied between plan and apply. | Re-run the build (new plan, new approval). |
| `unstash` fails: `No such saved stash 'plan'` | Different build, or the plan stage was skipped. | Stash and unstash within the same build. |
| Secrets left on agents | Workspaces not cleaned. | `cleanWs()` in `post { always }`. |

---

## Quick recap

- Same flow as topic 57: checks → plan (saved) → approval → apply the saved plan, one run per state.
- Jenkins tools: Docker agent with a pinned `hashicorp/terraform` image (`--entrypoint=`), multibranch for PR vs main, `when { branch 'main' }`, `input` with `submitter`, `stash`/`unstash`, `lock()`, `cleanWs()`.
- Credentials: agent **instance roles** (read-only plan agents, restricted apply agents) > assumed roles > stored keys via the AWS Credentials binding.

### Interview questions

1. **Q: How would you run Terraform safely in Jenkins?**
   A: A multibranch declarative pipeline in a pinned Terraform Docker image: fmt/validate/lint and a saved plan on every branch; on main only, an `input` approval restricted to a group, then apply the stashed plan. `lock()` per state, `-lock-timeout`, and `cleanWs()`. AWS access from agent instance roles: read-only for plan agents, write only for restricted apply agents.
2. **Q: Why use `agent none` on the approval stage?**
   A: An `input` step can wait for hours; with no agent allocated, it doesn't tie up an executor (or hold a lock) while waiting for a human.
3. **Q: How do you make sure the applied plan matches the approved one in Jenkins?**
   A: Save it with `plan -out=tfplan`, stash it (and archive the human-readable plan for the approver), then unstash and run `terraform apply tfplan` in the same build after approval. Terraform rejects it if the state changed.
