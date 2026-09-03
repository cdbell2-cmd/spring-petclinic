# Proofpoint SLSA L1 POC — Minimum File Set

This branch (`proofpoint-minimum`) contains only the files required to run the SLSA Level 1 POC pipeline on a Proofpoint CloudBees CI controller. Everything else has been removed to make the scope of the POC explicit.

For the full lab guide, see `SLSA-L1-POC-Proofpoint.md` at the repo root of the parent training directory.

---

## Why these files and not others

The Jenkinsfile runs three Maven commands:

```
./mvnw -B -DskipTests package org.cyclonedx:cyclonedx-maven-plugin:2.8.0:makeAggregateBom
./mvnw -B test
provenanceRecorder + archiveArtifacts  (Jenkins plugin steps — no file dependency)
```

Every file kept here is required by one of those commands.

---

## Files kept and why

| File / Directory | Why required |
|---|---|
| `Jenkinsfile` | Pipeline definition — the L1 requirement that the build is fully scripted and lives in SCM |
| `pom.xml` | Maven build descriptor; defines artifact coordinates (`spring-petclinic-4.0.0-SNAPSHOT.jar`), binds checkstyle and javaformat at the `validate` phase |
| `mvnw` | Maven wrapper shell script — must be executable (`git update-index --chmod=+x mvnw`); the Jenkinsfile calls `./mvnw` directly |
| `.mvn/wrapper/maven-wrapper.properties` | Tells `mvnw` which Maven version to download and from where |
| `src/checkstyle/nohttp-checkstyle.xml` | Required by `maven-checkstyle-plugin` (bound to `validate` phase in `pom.xml`); build fails at `validate` if the file is missing |
| `src/checkstyle/nohttp-checkstyle-suppressions.xml` | Suppression list referenced by `nohttp-checkstyle.xml` via `config_loc`; same failure mode if absent |
| `src/main/java/**` | Application Java source — compiled by `./mvnw package` |
| `src/main/resources/**` | Spring Boot resources (application.properties, Thymeleaf templates, static CSS/images, DB scripts) — packaged into the JAR |
| `src/test/java/**` | Test source — compiled and run by `./mvnw test` |

---

## Files removed and why

| File / Directory | Why removed |
|---|---|
| `mvnw.cmd` | Windows Maven wrapper — not needed on the Linux Kubernetes pod or EC2 (Linux AMI) agents used in the POC |
| `src/main/scss/` | SCSS source files — only compiled when Maven is invoked with `-Pcss`; the Jenkinsfile does not use that profile; the pre-compiled CSS is already present in `src/main/resources/static/resources/css/` |
| `src/test/jmeter/` | JMeter test plan — not executed by Maven Surefire; `./mvnw test` does not run it |
| `.devcontainer/` | VS Code dev container config — local development tooling only |
| `.editorconfig` | Editor formatting hints — no effect on CI builds |
| `.gitattributes` | Git line-ending rules — not needed on Linux build agents |
| `.github/` | GitHub Actions workflows — the POC uses Jenkins, not GitHub Actions |
| `.gitpod.yml` | Gitpod workspace config — irrelevant to CI |
| `build.gradle` / `gradlew*` / `gradle/` / `settings.gradle` | Gradle build files — the POC uses Maven (`./mvnw`); these are never invoked |
| `docker-compose.yml` | Local development convenience — not used in the pipeline |
| `k8s/` | Kubernetes deployment manifests — deployment is out of scope for the L1 POC |
| `LICENSE.txt` | Legal document — not needed for the build |
| `README.md` | Original spring-petclinic README — replaced by this file |
| `SLSA-L1-POC.md` | Generic (non-Proofpoint) lab guide — use `SLSA-L1-POC-Proofpoint.md` instead |

---

## Confirming the `mvnw` executable bit

If you clone this branch and the wrapper is not executable, fix it once:

```bash
git update-index --chmod=+x mvnw
git commit -m "make mvnw executable"
git push
```

---

## Artifact produced by `./mvnw -B -DskipTests package`

```
target/spring-petclinic-4.0.0-SNAPSHOT.jar   ← fingerprinted and archived
target/bom.json / target/bom.xml             ← CycloneDX SBOM (archived)
slsa/*.intoto.jsonl                          ← SLSA v0.2 provenance attestation (archived)
```
