// SLSA Level 1 POC — complete Jenkinsfile (see SLSA-L1-POC-Proofpoint.md §3.3)
// Rename to "Jenkinsfile" at the repo root of the spring-petclinic fork.
// Requires on the controller: SLSA Provenance Attestation plugin (id: slsa).
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
                provenanceRecorder artifactFilter: 'target/spring-petclinic-*.jar',
                                   targetDirectory: 'target/slsa'
                archiveArtifacts artifacts: 'target/slsa/*.intoto.jsonl', fingerprint: true
            }
        }

    }

    post {
        success { echo "SLSA L1 build complete: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
        failure { echo "Build failed: check the stage logs above" }
    }
}
