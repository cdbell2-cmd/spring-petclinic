# SLSA Level 1 Implementation POC — Proofpoint

**Target:** SLSA Level 1 on one agreed nonprod controller by **end of August 2026** (per Success Path proposal, 08/04/26)
**Reference controller:** `buildrel` — `https://cloudbees.nonprod-cia-awsuse.nonprod.ppops.net/buildrel/`
**Platform:** CloudBees CI Modern on AWS EKS · Jenkins `2.541.1.35570` · CJOC-managed controller
**Build tool:** Maven (`./mvnw`) · **Example app:** spring-petclinic (stand-in for any Proofpoint Maven service)
**Time:** ~60 minutes

Read `SLSA-Feynman.md` first if you have not. SLSA Level 2 is covered separately in `SLSA-L2-POC-Proofpoint.md` — do not start it until every item in the L1 checklist at the bottom of this document is green.

---

## How this POC matches the Proofpoint environment

Everything below was verified from the two support bundles (controllers `buildrel` and `cloud15`, captured 2026-08-25). The lab adds exactly one plugin — the **SLSA Provenance Attestation** plugin (`slsa`, https://plugins.jenkins.io/slsa/), installed in Lab 3 — everything else is already in place.

| Environment fact (from support bundles) | How the lab uses it |
|---|---|
| Jobs are overwhelmingly Pipeline: 5,662 `WorkflowJob` + 90 Multibranch on `buildrel`; 1,462 + 151 on `cloud15` | POC is a Multibranch Pipeline with a source-controlled Jenkinsfile |
| GitHub Branch Source `1925.x`, `cloudbees-github-reporting`, plus Bitbucket Server integration installed | Lab 1 uses GitHub branch source; a Bitbucket note is included |
| Two agent models: Kubernetes pods (kubernetes plugin `4419.x`, namespace `cloudbees-agents`, service account `jenkins-agents`) **and** EC2 cloud `ec2-instances-cloud` (m6a.large / m5a.2xlarge / m5a.4xlarge, label-routed) | Primary Jenkinsfile uses a Kubernetes pod agent; §2.5 gives the EC2 label variant. The provenance stage is written to run on either |
| Default pod agent image is pulled through the internal mirror `repocache.nonprod.ppops.net/docker/...` (internal-only hostname) | The lab Jenkinsfile defaults to the public image so it runs anywhere; when running inside Proofpoint, switch it to the `repocache` mirror path so pulls follow their registry controls |
| The `slsa` plugin is **not** in either bundle's plugin list | Lab 3 installs it (v`40.v733b_0005fa_fd`, min Jenkins 2.361.4 — compatible with 2.541.1.35570). It is a community plugin outside the CloudBees Assurance Program: fine to install ad-hoc for the POC, but the fleet rollout must add it to the plugin catalog in the CJOC CasC bundle |
| `pipeline-utility-steps 2.20.0` installed | Provenance generation is handled by the SLSA plugin; `sha256()`/`findFiles` remain available for ad-hoc digest cross-checks |
| `artifact-manager-s3 962.x` installed; S3 blob store configured for bucket `nonprod.cloudbees-ci-cache.pfpt` — but the artifact-manager factory list in the bundle snapshot is **empty** | `archiveArtifacts` + `fingerprint: true` works identically whether artifacts land on controller disk or in S3. §3.5 tells you how to confirm which one is active |
| Fingerprint storage: local file storage (default), cleanup enabled | Fingerprints UI is used as the L1 audit trail |
| CasC bundles from CJOC (`core.casc.config.bundle`), RBAC (`nectar-rbac`), Folders Plus | Not needed for L1; used heavily in the L2 POC and the fleet-rollout runbook |

> **Note on anonymized names:** support bundles anonymize job names, labels, and agent names (e.g. `label_pleasant_rugby`). Where the lab says *"your EC2 label"* or *"your team folder"*, substitute the real value from the controller UI.

> **Image sourcing:** the lab Jenkinsfile defaults to the public `maven:3.9-eclipse-temurin-17` (Docker Hub) so it runs on any test controller. `repocache.nonprod.ppops.net` is Proofpoint's **internal-only** mirror — it does not resolve outside their network. When delivering inside Proofpoint, switch the image to the `repocache` path per the note under the Jenkinsfile. Everything else in the lab is environment-neutral.

---

## What SLSA Level 1 requires (and does not)

Level 1 = **the build process is fully scripted and produces provenance**. Concretely, for every build:

1. The pipeline definition lives in SCM (no UI-defined build steps).
2. The build runs on the hosted platform (CloudBees CI), not a laptop.
3. The build outputs a provenance document: what was built, from which commit, by which builder, and the artifact's digest.

No signing, no tamper-proofing — that is Level 2. L1 is the auditable paper trail.

```
Lab 1:  Create the Multibranch Pipeline job                     (~15 min)
Lab 2:  Base Jenkinsfile — Maven build + test                   (~20 min)
Lab 3:  L1 provenance — SLSA plugin attestation + SBOM          (~25 min)
```

---

## Pre-lab — repo and access

1. Fork https://github.com/spring-projects/spring-petclinic (or pick a real Proofpoint Maven service — any repo with `pom.xml` + `mvnw` works; adjust artifact paths accordingly).
2. Confirm you can log in to the agreed nonprod controller (SAML SSO) and can create items in a test folder.
3. (Proofpoint delivery only) Confirm build agents can reach the internal registry mirror `repocache.nonprod.ppops.net` — already the default for pod agents. On a test controller outside Proofpoint, skip this; the lab uses public images.

After `./mvnw package` the example app produces:

```
target/spring-petclinic-4.0.0-SNAPSHOT.jar   ← the artifact we fingerprint
target/bom.json / target/bom.xml             ← CycloneDX SBOM
```

> For Proofpoint's own services whose poms do **not** declare the CycloneDX plugin, no pom change is needed — invoke it by full coordinates:
> `./mvnw -B org.cyclonedx:cyclonedx-maven-plugin:2.8.0:makeAggregateBom`
> (the plugin resolves through the existing Artifactory/`repocache` Maven mirror).

---

## Lab 1 — Create the Multibranch Pipeline

**Goal:** Connect the controller to the fork so every push triggers a build. This satisfies the "hosted build service" foundation of SLSA.

### Steps

**1.1 Credential**

1. On the controller: **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**
2. Kind **Username with password** — GitHub username + a classic PAT with the full **`repo`** scope
3. ID: `github-slsa-poc`

> Proofpoint also runs Bitbucket Server integration on this controller. If the pilot repo lives in Bitbucket, use **Bitbucket** as the branch source in 1.2 with the existing Bitbucket endpoint; everything else in this POC is identical.

**1.2 Job**

1. In your team/test folder: **New Item** → name `slsa-poc-petclinic` → **Multibranch Pipeline**
2. **Branch Sources → Add source → GitHub**: credential `github-slsa-poc`, repository HTTPS URL of your fork
3. **Build Configuration**: Mode = by Jenkinsfile, Script Path = `Jenkinsfile`
4. Save — the scan runs and reports `'Jenkinsfile' not found`, which is expected until Lab 2.

### Expected result

Multibranch job exists inside a folder, scan completes successfully with no buildable branches yet.

---

## Lab 2 — Base Jenkinsfile (matches the Proofpoint agent model)

**Goal:** A scripted, source-controlled Maven build — the core L1 requirement.

### 2.1 The Jenkinsfile — Kubernetes pod agent (primary)

This mirrors how pod agents already run on `buildrel` (namespace `cloudbees-agents`, service account `jenkins-agents` — both injected by the controller's defaults, so the Jenkinsfile does not declare them). Create `Jenkinsfile` at the repo root:

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    # Public image (Docker Hub). Inside Proofpoint, switch to the internal
    # mirror: repocache.nonprod.ppops.net/docker/maven:3.9-eclipse-temurin-17
    # (repocache is internal-only; it does not resolve outside their network)
    image: maven:3.9-eclipse-temurin-17
    command: [sleep, infinity]
    env:
    - name: MAVEN_OPTS
      value: "-Xmx512m"
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  volumes:
  - name: maven-cache
    emptyDir: {}
"""
        }
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHA = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    env.GIT_REPO_URL   = sh(script: 'git remote get-url origin', returnStdout: true).trim()
                    echo "Building commit: ${env.GIT_COMMIT_SHA}"
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    // makeAggregateBom invoked explicitly so it works with or
                    // without the plugin being declared in the app's pom.xml
                    sh './mvnw -B -DskipTests package org.cyclonedx:cyclonedx-maven-plugin:2.8.0:makeAggregateBom'
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh './mvnw -B test'
                }
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

    }

    post {
        success { echo "Build succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
        failure { echo "Build failed: check the stage logs above" }
    }
}
```

Notes on why this differs from a generic lab Jenkinsfile:

- **Image source** — the lab default is the public `maven:3.9-eclipse-temurin-17` so the pipeline runs anywhere. When running inside Proofpoint, switch to `repocache.nonprod.ppops.net/docker/maven:3.9-eclipse-temurin-17` — the same mirror path the controller already uses for its default agent image (`.../docker/cloudbees/cloudbees-core-agent`). Docker Hub pulls may be blocked there and would bypass Proofpoint's registry controls; conversely, `repocache` is internal-only and unreachable from any other network.
- **No `namespace` or `serviceAccountName` in the pod YAML — deliberately.** The Proofpoint controllers already inject the right defaults via JVM flags (`-Dcom.cloudbees.jenkins.plugins.kube.NamespaceFilter.defaultNamespace=cloudbees-agents`, `ServiceAccountFilter.defaultServiceAccount=jenkins-agents`), so pods land in `cloudbees-agents` as `jenkins-agents` without the Jenkinsfile saying so. Hardcoding a namespace makes the pipeline fail with `pods is forbidden ... 403` on any controller whose service account has RBAC in a different namespace — omitting it keeps the same Jenkinsfile working on all ~70 Proofpoint controllers *and* on scratch/test clusters.

### 2.5 Variant — EC2 agent

Many Proofpoint workloads run on the `ec2-instances-cloud` EC2 templates rather than pods. The same pipeline works by swapping the agent block and dropping the `container()` wrappers:

```groovy
agent { label 'YOUR-EC2-TEMPLATE-LABEL' }   // e.g. the m6a.large standard-network label
```

Requirements on the AMI (`ami-0408a16db20ae3c94` family): Java 17+ available for `./mvnw`. Everything in Lab 3 is agent-agnostic — it uses Jenkins steps (`provenanceRecorder`, `archiveArtifacts`), not shell utilities, precisely so it behaves identically on pods and EC2.

### 2.6 Commit, push, build

```bash
git add Jenkinsfile
git commit -m "SLSA L1 POC: scripted Maven build"
git push origin main
```

Trigger **Scan Multibranch Pipeline Now**; the `main` pipeline appears and builds. First build takes 3–5 minutes (cold Maven cache).

### Expected result

Green build with Checkout / Build / Test stages; log shows the pod (or EC2 agent) provisioning, `./mvnw` producing the JAR, and tests passing.

---

## Lab 3 — SLSA Level 1 provenance (SLSA Provenance Attestation plugin)

**Goal:** Every build archives a fingerprinted JAR, an SBOM, and a machine-generated in-toto/SLSA provenance attestation tying artifact digest → commit → builder → build run. Provenance is produced by the Jenkins **SLSA Provenance Attestation** plugin, not hand-rolled JSON — the format is standard, and there is no string-templating to get wrong.

### 3.0 Install the SLSA Provenance Attestation plugin

1. On the controller: **Manage Jenkins → Plugins → Available plugins** → search **SLSA** → install **SLSA Provenance Attestation** (`slsa`). No restart required.
2. Version at time of writing: `40.v733b_0005fa_fd`; requires Jenkins ≥ 2.361.4 — satisfied by the fleet's 2.541.1.35570.

> **CAP note:** this is a community plugin outside the CloudBees Assurance Program, and the controllers are CasC-managed with the non-CAP plugin monitor active. Ad-hoc install is fine for the POC; the fleet rollout must add `slsa` to the **plugin catalog** in the CJOC CasC bundle so Beekeeper does not flag or remove it (see the L2 POC appendix).

What the plugin emits (verified from the plugin source, not just its docs):

- One in-toto attestation per matched artifact, written as `<artifact-name>.intoto.jsonl` into the target directory (`multiple.intoto.jsonl` if several artifacts match one attestation).
- The predicate is **SLSA Provenance v0.2**, containing: `subject[]` with the artifact's SHA-256; `invocation.configSource` = git repo URL + commit digest + job name as entryPoint; `invocation.environment` = `job_url`, `build_url`, `node_name`; `materials` = the source repo at the built commit; `metadata` = build number as `buildInvocationId` plus build start/finish timestamps.
- Known limitations: only Git SCM is supported (fine — the POC checks out over git from GitHub/Bitbucket), executed build steps are not recorded in the attestation, and **the plugin does not sign attestations**. Signing is exactly what the L2 POC adds with cosign.

### 3.1 Add the Archive stage

After `Test`:

```groovy
stage('Archive') {
    steps {
        // fingerprint: true records the artifact hash in the Jenkins
        // fingerprint database — traceable across jobs via copyartifact
        archiveArtifacts artifacts: 'target/spring-petclinic-*.jar', fingerprint: true
        archiveArtifacts artifacts: 'target/bom.json, target/bom.xml', allowEmptyArchive: true
    }
}
```

### 3.2 Add the provenance stage

`provenanceRecorder` is a regular pipeline step (the plugin implements `SimpleBuildStep`), so it runs as its own stage on either agent type — no container wrapper, no shell utilities, identical on pods and EC2:

```groovy
stage('SLSA L1 Provenance') {
    steps {
        script {
            // provenanceRecorder generates attestations only when the build
            // result is already SUCCESS; mid-pipeline it is still null, so
            // set it explicitly (Jenkins only allows it to worsen afterward)
            currentBuild.result = 'SUCCESS'
        }
        // targetDirectory is workspace-root 'slsa', NOT 'target/slsa' — see below
        provenanceRecorder artifactFilter: 'target/spring-petclinic-*.jar',
                           targetDirectory: 'slsa'
        archiveArtifacts artifacts: 'slsa/*.intoto.jsonl', fingerprint: true
    }
}
```

Three placement rules, all verified against the plugin source or hit in dry runs:

- **The `currentBuild.result = 'SUCCESS'` line is required.** The plugin's `perform()` begins with `if (run.getResult() != Result.SUCCESS) { skip }`, and in a running Pipeline `getResult()` stays `null` until the build finishes — so without the explicit set, the step logs `[slsa] - build not successful, not generating provenance attestations` and writes nothing. Setting SUCCESS mid-build is safe: Jenkins only permits the result to get worse afterward, so later failures still fail the build.
- **`targetDirectory` must be a workspace-root path (`slsa`), not under `target/`.** The recorder executes in the jnlp container as user `jenkins` (uid 1000), while the maven container runs as root and created `target/` root-owned — the plugin's `mkdirs` under `target/` then fails with `AccessDeniedException`. The workspace root is owned by the jnlp user, so `slsa/` is always writable. (Reading the root-owned JAR works — root's files are world-readable.)
- The stage must run **after** `Build` (and after `Archive` in this layout) — the JAR has to exist in the workspace for the `artifactFilter` glob to match.

### 3.3 Your complete Jenkinsfile after Lab 3

This is the full file — Lab 2 base plus the two Lab 3 stages. It is the L1 deliverable:

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    # Public image (Docker Hub). Inside Proofpoint, switch to the internal
    # mirror: repocache.nonprod.ppops.net/docker/maven:3.9-eclipse-temurin-17
    # (repocache is internal-only; it does not resolve outside their network)
    image: maven:3.9-eclipse-temurin-17
    command: [sleep, infinity]
    env:
    - name: MAVEN_OPTS
      value: "-Xmx512m"
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  volumes:
  - name: maven-cache
    emptyDir: {}
"""
        }
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHA = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    env.GIT_REPO_URL   = sh(script: 'git remote get-url origin', returnStdout: true).trim()
                    echo "Building commit: ${env.GIT_COMMIT_SHA}"
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    // makeAggregateBom invoked explicitly so it works with or
                    // without the plugin being declared in the app's pom.xml
                    sh './mvnw -B -DskipTests package org.cyclonedx:cyclonedx-maven-plugin:2.8.0:makeAggregateBom'
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh './mvnw -B test'
                }
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('Archive') {
            steps {
                // fingerprint: true records the artifact hash in the Jenkins
                // fingerprint database — traceable across jobs via copyartifact
                archiveArtifacts artifacts: 'target/spring-petclinic-*.jar', fingerprint: true
                archiveArtifacts artifacts: 'target/bom.json, target/bom.xml', allowEmptyArchive: true
            }
        }

        stage('SLSA L1 Provenance') {
            steps {
                script {
                    // provenanceRecorder generates attestations only when the build
                    // result is already SUCCESS; mid-pipeline it is still null, so
                    // set it explicitly (Jenkins only allows it to worsen afterward)
                    currentBuild.result = 'SUCCESS'
                }
                // targetDirectory is workspace-root 'slsa', NOT 'target/slsa':
                // the recorder runs in the jnlp container as user 'jenkins',
                // while the maven container created target/ as root — mkdirs
                // under target/ fails with AccessDeniedException
                provenanceRecorder artifactFilter: 'target/spring-petclinic-*.jar',
                                   targetDirectory: 'slsa'
                archiveArtifacts artifacts: 'slsa/*.intoto.jsonl', fingerprint: true
            }
        }

    }

    post {
        success { echo "SLSA L1 build complete: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
        failure { echo "Build failed: check the stage logs above" }
    }
}
```

### 3.4 Commit, push, run

```bash
git add Jenkinsfile
git commit -m "SLSA L1: fingerprinting, SBOM, plugin-generated provenance attestation"
git push origin main
```

### 3.5 Inspect the results

1. **Archived Artifacts** on the build page: the JAR, `bom.json`, `bom.xml`, and `spring-petclinic-4.0.0-SNAPSHOT.jar.intoto.jsonl`.
   - *Where did they land?* The `artifact-manager-s3` plugin is installed and the blob store points at the `nonprod.cloudbees-ci-cache.pfpt` S3 bucket, but the bundle snapshot shows **no active artifact-manager factory**, which means artifacts are currently written to controller disk. Check **Manage Jenkins → System → Artifact Management for Builds**. Either backend satisfies L1; if Proofpoint enables S3 later, nothing in this pipeline changes.
2. Download the `.intoto.jsonl` file and inspect it. It is an in-toto envelope whose `payload` is the base64-encoded provenance statement — decode it:

   ```bash
   jq -r '.payload' spring-petclinic-4.0.0-SNAPSHOT.jar.intoto.jsonl | base64 -d | jq .
   ```

   (If the file already shows plain JSON with a top-level `predicate`, just `jq .` it directly.) Confirm:
   - `subject[0].digest.sha256` is present,
   - `predicate.invocation.configSource.digest` matches the commit SHA in the build log,
   - `predicate.invocation.environment.build_url` points at this exact run.
3. **Fingerprints** in the build's left nav — the JAR and the attestation both appear. This database is what lets an auditor answer "which build produced this exact JAR?" months later.

### Expected result

Five-stage green build. Every subsequent build of any branch automatically produces artifact + SBOM + a standards-format SLSA v0.2 attestation, with no human step in the loop.

---

## SLSA Level 1 completion checklist

- [ ] Jenkinsfile in SCM; no UI-defined build steps (Lab 2)
- [ ] Build runs on the CloudBees CI controller via pod or EC2 agent — never locally (Lab 2)
- [ ] JAR archived with `fingerprint: true` (Lab 3)
- [ ] CycloneDX SBOM (`bom.json`/`bom.xml`) archived per build (Lab 3)
- [ ] SLSA Provenance Attestation plugin installed; `.intoto.jsonl` (SLSA v0.2 predicate) archived per build with commit SHA + artifact SHA-256 (Lab 3)
- [ ] Fingerprints visible in the Jenkins Fingerprints UI (Lab 3)
- [ ] Verified whether archived artifacts land on controller disk or the S3 bucket (§3.5)

**Definition of done for the proposal milestone:** one Proofpoint Maven pipeline on the agreed controller runs this end-to-end, and the team can locate the provenance for any given build in under a minute.

Proceed to `SLSA-L2-POC-Proofpoint.md` (target: September 22) to add signing.

---

## Troubleshooting

**Pod never starts / `ErrImagePull` on the maven container**
- Outside Proofpoint: the image path must be the public `maven:3.9-eclipse-temurin-17` — the `repocache.nonprod.ppops.net` mirror is internal-only and unreachable (it resolves to a sinkhole publicly).
- Inside Proofpoint: use `repocache.nonprod.ppops.net/docker/maven:3.9-eclipse-temurin-17`; if the mirror does not yet cache this image, ask the registry team to whitelist/prefetch it — do not fall back to Docker Hub there.

**`ERROR: Failed to launch ...` with `pods is forbidden: User "system:serviceaccount:<ns>:<controller>" cannot list resource "pods" in ... namespace "..."` (403)**
The pod YAML names a namespace the controller's Kubernetes service account has no RBAC for. Remove `metadata.namespace` (and `serviceAccountName`) from the pod YAML and let the controller's cloud defaults apply — on the Proofpoint controllers the CloudBees filters inject `cloudbees-agents`/`jenkins-agents` automatically. Only hardcode a namespace if the platform team has granted the controller RBAC there.

**`Invalid option type "timestamps". Valid option types: [buildDiscarder, ...]` at pipeline startup**
Someone added `timestamps()` to the `options {}` block on a controller without the Timestamper plugin — Declarative validates options against installed plugins at compile time, so the build fails before any stage runs. Remove the option or install Timestamper (both audited Proofpoint controllers have it; a scratch/test controller may not). The lab Jenkinsfile deliberately omits it.

**`Could not update commit status ... "Resource not accessible by personal access token" 403`**
The GitHub token cannot write commit statuses. This does not fail the build — it only stops CloudBees CI posting ✓/✗ back to GitHub. That exact message means a **fine-grained** PAT is in use: grant it **Commit statuses: Read and write** on the repo (or use a classic PAT with the full `repo` scope as in step 1.1). A classic PAT missing `repo:status` produces a plain 403 with the same fix.

**`./mvnw: Permission denied`**
`git update-index --chmod=+x mvnw && git commit -m "make mvnw executable" && git push`

**`No such DSL method 'provenanceRecorder'`**
The SLSA Provenance Attestation plugin is not installed on this controller (§3.0). On CasC-managed controllers, also confirm `slsa` is in the plugin catalog — otherwise Beekeeper can remove it at the next bundle reload.

**`AccessDeniedException: .../target/slsa` (or any path under `target/`) from `provenanceRecorder`**
The recorder runs in the jnlp container as user `jenkins`, but `target/` was created by the maven container running as root — the plugin cannot `mkdirs` inside it. Set `targetDirectory` to a workspace-root path (`slsa`, as in §3.2), which the jnlp user owns. The same uid mismatch applies to any step outside `container('maven')` that writes under `target/`.

**`[slsa] - build not successful, not generating provenance attestations` (and the follow-on archive step fails with `'slsa/*.intoto.jsonl' doesn't match anything`)**
The plugin only generates attestations when `run.getResult()` is already `SUCCESS`, and in a running Pipeline the result is `null` until the build ends. Add `script { currentBuild.result = 'SUCCESS' }` immediately before `provenanceRecorder` (as in §3.2). This is safe — Jenkins only allows the result to be downgraded afterward, so later failures still fail the build.

**No `.intoto.jsonl` produced, or `multiple.intoto.jsonl` instead of one per JAR**
The `artifactFilter` glob is workspace-relative and must match after the `Build` stage has produced the JAR. One matching file → `<jar-name>.intoto.jsonl`; several matches → a combined `multiple.intoto.jsonl`. Tighten the glob if you want one attestation per artifact.

**Attestation shows the wrong repo or no source info**
The plugin only supports the Git SCM provider — it reads the repo URL and commit from the build's git checkout. Ensure the job checks out via `checkout scm` (Multibranch does) rather than a shallow custom fetch.

**Maven cannot resolve `cyclonedx-maven-plugin`**
The agent is not using the Artifactory mirror. Check `~/.m2/settings.xml` provisioning (config-file-provider is installed if a managed settings file is preferred: `withMaven` or `configFileProvider` can inject it).

**Builds queue forever on the EC2 variant**
The label must match an `ec2-instances-cloud` template exactly (labels in the support bundle are anonymized — read the real ones from **Manage Jenkins → Clouds → ec2-instances-cloud**).
