# SLSA Level 1 POC — Proofpoint (spring-petclinic)

This branch (`proofpoint-minimum`) contains the minimum files needed to run the SLSA Level 1 POC on a Proofpoint CloudBees CI controller. See `README-proofpoint-minimum.md` for the rationale behind what was kept and what was removed.

**Target:** SLSA Level 1 on one agreed nonprod controller by end of August 2026
**Reference controller:** `buildrel` — `https://cloudbees.nonprod-cia-awsuse.nonprod.ppops.net/buildrel/`
**Platform:** CloudBees CI Modern on AWS EKS · Jenkins `2.541.1.35570` · CJOC-managed controller
**Build tool:** Maven (`./mvnw`) · **App:** spring-petclinic (stand-in for any Proofpoint Maven service)
**Estimated time:** ~60 minutes across three labs

---

## What SLSA Level 1 requires

Level 1 = **the build process is fully scripted and produces provenance**. For every build:

1. The pipeline definition lives in SCM (no UI-defined build steps).
2. The build runs on the hosted platform (CloudBees CI), not a laptop.
3. The build outputs a provenance document: what was built, from which commit, by which builder, and the artifact's digest.

No signing, no tamper-proofing — that is Level 2.

```
Lab 1:  Create the Multibranch Pipeline job                     (~15 min)
Lab 2:  Verify the Jenkinsfile — Maven build + test             (~20 min)
Lab 3:  L1 provenance — SLSA plugin attestation + SBOM          (~25 min)
```

After `./mvnw package` this app produces:

```
target/spring-petclinic-4.0.0-SNAPSHOT.jar   ← artifact we fingerprint
target/bom.json / target/bom.xml             ← CycloneDX SBOM
slsa/*.intoto.jsonl                          ← SLSA v0.2 provenance attestation
```

---

## Pre-lab — access and repo setup

1. **Fork or use this repo directly.** Any Maven repo with `pom.xml` + `mvnw` works; adjust the artifact glob in the Jenkinsfile (`target/spring-petclinic-*.jar`) to match your artifact name.
2. **Controller access.** Confirm you can log in to the agreed nonprod controller via SAML SSO and can create items in a test folder.
3. **Registry access (Proofpoint delivery only).** Confirm build agents can reach `repocache.nonprod.ppops.net`. The Jenkinsfile defaults to the public `maven:3.9-eclipse-temurin-17` (Docker Hub) so it runs anywhere; when delivering inside Proofpoint, switch the image to the `repocache` mirror path (see the comment in the Jenkinsfile).
4. **`mvnw` executable bit.** If cloning on a system that strips execute permissions, run once:
   ```bash
   git update-index --chmod=+x mvnw
   git commit -m "make mvnw executable"
   git push
   ```

---

## Lab 1 — Create the Multibranch Pipeline

**Goal:** Connect the controller to this repo so every push triggers a build. This satisfies the "hosted build service" foundation of SLSA.

### 1.1 Credential

1. On the controller: **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**
2. Kind **Username with password** — GitHub username + a classic PAT with the full **`repo`** scope
3. ID: `github-slsa-poc`

> If the pilot repo lives in Bitbucket Server, use **Bitbucket** as the branch source in step 1.2 with the existing Bitbucket endpoint; everything else in this POC is identical.

### 1.2 Job

1. In your team/test folder: **New Item** → name `slsa-poc-petclinic` → **Multibranch Pipeline**
2. **Branch Sources → Add source → GitHub**: credential `github-slsa-poc`, repository HTTPS URL of this repo
3. **Build Configuration**: Mode = by Jenkinsfile, Script Path = `Jenkinsfile`
4. Under **Branch Sources**, set the branch filter to include the `proofpoint-minimum` branch (or `main` if you have merged the Jenkinsfile there).
5. Save — the scan runs and discovers the branch.

### Expected result

Multibranch job exists inside a folder; scan completes and discovers the branch with the Jenkinsfile. The first build is queued automatically.

---

## Lab 2 — Verify the Jenkinsfile

**Goal:** Confirm the scripted, source-controlled Maven build runs end-to-end. The Jenkinsfile is already in this repo at the root.

### Agent model

The Jenkinsfile uses a Kubernetes pod agent — the primary model on `buildrel` (namespace `cloudbees-agents`, service account `jenkins-agents` injected by the controller's JVM defaults; the Jenkinsfile does not declare them to stay portable across all ~70 Proofpoint controllers).

**EC2 variant:** many Proofpoint workloads use the `ec2-instances-cloud` EC2 templates. Swap the `agent` block to:

```groovy
agent { label 'YOUR-EC2-TEMPLATE-LABEL' }   // e.g. the m6a.large standard-network label
```

Drop the `container('maven') { }` wrappers — Maven is available directly on the AMI. The provenance stage is agent-agnostic.

### Image note

| Environment | Image to use |
|---|---|
| Any test controller (outside Proofpoint) | `maven:3.9-eclipse-temurin-17` (Docker Hub — already in the Jenkinsfile) |
| Inside Proofpoint | `repocache.nonprod.ppops.net/docker/maven:3.9-eclipse-temurin-17` |

`repocache` is internal-only and does not resolve outside Proofpoint's network.

### Expected result

Green build with Checkout / Build / Test stages. Log shows the pod (or EC2 agent) provisioning, `./mvnw` producing the JAR, and tests passing. First build takes 3–5 minutes (cold Maven cache).

---

## Lab 3 — SLSA Level 1 provenance

**Goal:** Every build archives a fingerprinted JAR, an SBOM, and a machine-generated SLSA v0.2 attestation tying artifact digest → commit → builder → build run.

### 3.0 Install the SLSA Provenance Attestation plugin

1. **Manage Jenkins → Plugins → Available plugins** → search **SLSA** → install **SLSA Provenance Attestation** (plugin id: `slsa`). No restart required.
2. Version at time of writing: `40.v733b_0005fa_fd`; requires Jenkins ≥ 2.361.4 — satisfied by the fleet's 2.541.1.35570.

> **CAP note:** this plugin is outside the CloudBees Assurance Program. Ad-hoc install is fine for the POC; the fleet rollout must add `slsa` to the **plugin catalog** in the CJOC CasC bundle so Beekeeper does not flag or remove it at the next bundle reload.

**What the plugin emits:**

- One in-toto attestation per matched artifact: `<artifact-name>.intoto.jsonl` in `targetDirectory`.
- Predicate is **SLSA Provenance v0.2**, containing:
  - `subject[].digest.sha256` — artifact SHA-256
  - `invocation.configSource` — git repo URL + commit digest + job name
  - `invocation.environment` — `job_url`, `build_url`, `node_name`
  - `materials` — source repo at the built commit
  - `metadata` — `buildInvocationId` (build number) + start/finish timestamps
- Known limitations: only Git SCM is supported; executed build steps are not recorded; **attestations are not signed** (signing is what L2 adds with cosign).

### 3.1 Verify the Archive stage

The Jenkinsfile already contains this stage (no changes needed):

```groovy
stage('Archive') {
    steps {
        archiveArtifacts artifacts: 'target/spring-petclinic-*.jar', fingerprint: true
        archiveArtifacts artifacts: 'target/bom.json, target/bom.xml', allowEmptyArchive: true, fingerprint: true
    }
}
```

`fingerprint: true` records the artifact hash in the Jenkins fingerprint database — the audit trail that lets an auditor answer "which build produced this exact JAR?" months later.

### 3.2 Verify the provenance stage

The Jenkinsfile already contains this stage (no changes needed):

```groovy
stage('SLSA L1 Provenance') {
    steps {
        script {
            currentBuild.result = 'SUCCESS'
        }
        provenanceRecorder artifactFilter: 'target/spring-petclinic-*.jar',
                           targetDirectory: 'slsa'
        archiveArtifacts artifacts: 'slsa/*.intoto.jsonl', fingerprint: true
    }
}
```

Three non-obvious rules baked into this stage:

| Rule | Why |
|---|---|
| `currentBuild.result = 'SUCCESS'` before `provenanceRecorder` | The plugin's `perform()` skips if `run.getResult() != SUCCESS`. In a running pipeline `getResult()` returns `null`, so without the explicit set the step logs `build not successful` and writes nothing. Setting SUCCESS is safe — Jenkins only allows the result to worsen afterward. |
| `targetDirectory: 'slsa'` (workspace root), not `target/slsa` | The recorder runs in the `jnlp` container as user `jenkins` (uid 1000). The `maven` container ran as root and owns `target/`; `mkdirs` under it fails with `AccessDeniedException`. The workspace root is owned by the `jnlp` user. |
| Stage runs after `Build` (and after `Archive`) | The JAR must exist in the workspace for the `artifactFilter` glob to match. |

### 3.3 Inspect the results after a build

1. **Archived Artifacts** on the build page: the JAR, `bom.json`, `bom.xml`, and `spring-petclinic-4.0.0-SNAPSHOT.jar.intoto.jsonl`.

2. **Decode and inspect the attestation:**
   ```bash
   jq -r '.payload' spring-petclinic-4.0.0-SNAPSHOT.jar.intoto.jsonl | base64 -d | jq .
   ```
   Confirm:
   - `subject[0].digest.sha256` is present
   - `predicate.invocation.configSource.digest` matches the commit SHA in the build log
   - `predicate.invocation.environment.build_url` points at this exact run

3. **Fingerprints** in the build's left nav — the JAR and the attestation both appear.

4. **Artifact storage:** check **Manage Jenkins → System → Artifact Management for Builds**. The `artifact-manager-s3` plugin is installed and the blob store points at `nonprod.cloudbees-ci-cache.pfpt`, but the bundle snapshot shows no active artifact-manager factory — artifacts currently land on controller disk. Either backend satisfies L1; if Proofpoint enables S3 later, nothing in this pipeline changes.

### Expected result

Five-stage green build: Checkout → Build → Test → Archive → SLSA L1 Provenance. Every subsequent build of any branch automatically produces artifact + SBOM + standards-format SLSA v0.2 attestation with no human step in the loop.

---

## SLSA Level 1 completion checklist

- [ ] Jenkinsfile in SCM; no UI-defined build steps
- [ ] Build runs on the CloudBees CI controller via pod or EC2 agent — never locally
- [ ] JAR archived with `fingerprint: true`
- [ ] CycloneDX SBOM (`bom.json`/`bom.xml`) archived per build
- [ ] SLSA Provenance Attestation plugin (`slsa`) installed on the controller
- [ ] `.intoto.jsonl` (SLSA v0.2 predicate) archived per build — commit SHA + artifact SHA-256 both present
- [ ] Fingerprints visible in the Jenkins Fingerprints UI
- [ ] Verified whether archived artifacts land on controller disk or S3 (§3.3 step 4)

**Definition of done:** one Proofpoint Maven pipeline on the agreed controller runs this end-to-end, and the team can locate the provenance for any given build in under a minute.

---

## Troubleshooting

**Pod never starts / `ErrImagePull` on the maven container**
- Outside Proofpoint: use the public `maven:3.9-eclipse-temurin-17` — the `repocache` mirror is internal-only.
- Inside Proofpoint: use `repocache.nonprod.ppops.net/docker/maven:3.9-eclipse-temurin-17`; if the image is not yet cached, ask the registry team to whitelist/prefetch it.

**`pods is forbidden ... 403` at pod startup**
The pod YAML names a namespace the controller's service account has no RBAC for. Remove `metadata.namespace` (and `serviceAccountName`) from the pod spec — the Proofpoint controllers inject `cloudbees-agents`/`jenkins-agents` automatically via JVM flags.

**`Invalid option type "timestamps"` at pipeline startup**
The `timestamps()` option requires the Timestamper plugin. The Jenkinsfile deliberately omits it for portability; add it only if both the target controller and any scratch/test controllers have the plugin.

**`Could not update commit status ... 403` (GitHub)**
The PAT cannot write commit statuses. Does not fail the build — only stops CI from posting ✓/✗ to GitHub. Fix: grant the PAT **Commit statuses: Read and write** (fine-grained) or use a classic PAT with full `repo` scope.

**`./mvnw: Permission denied`**
```bash
git update-index --chmod=+x mvnw && git commit -m "make mvnw executable" && git push
```

**`No such DSL method 'provenanceRecorder'`**
The SLSA Provenance Attestation plugin is not installed on this controller. On CasC-managed controllers, also confirm `slsa` is in the plugin catalog — otherwise Beekeeper removes it at the next bundle reload.

**`AccessDeniedException` under `target/` from `provenanceRecorder`**
The recorder runs as user `jenkins` but `target/` is root-owned (created by the maven container). Set `targetDirectory: 'slsa'` (workspace root) — the `jnlp` user owns it. See §3.2.

**`[slsa] - build not successful, not generating provenance attestations`**
Add `script { currentBuild.result = 'SUCCESS' }` immediately before `provenanceRecorder`. The plugin skips if `getResult()` is not `SUCCESS`, and in a running pipeline it is `null`. See §3.2.

**`'slsa/*.intoto.jsonl' doesn't match anything` (archive step after provenance)**
Usually caused by the `[slsa] - build not successful` skip above. Fix: add the `currentBuild.result` line. If the plugin ran but produced `multiple.intoto.jsonl` instead of one per JAR, tighten the `artifactFilter` glob.

**Attestation shows wrong repo or no source info**
The plugin only supports the Git SCM provider. Ensure the job checks out via `checkout scm` (Multibranch does this by default).

**Maven cannot resolve `cyclonedx-maven-plugin`**
The agent is not using the Artifactory mirror. Check `~/.m2/settings.xml` provisioning — `config-file-provider` is installed; use `withMaven` or `configFileProvider` to inject a managed settings file that points at the internal Artifactory mirror.

**Builds queue forever on the EC2 variant**
The label must match an `ec2-instances-cloud` template exactly. Read the real labels from **Manage Jenkins → Clouds → ec2-instances-cloud** — support bundle labels are anonymized.
