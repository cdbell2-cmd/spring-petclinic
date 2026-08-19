pipeline {
   agent {
    kubernetes {
        yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: [sleep, infinity]
    env:
    - name: MAVEN_OPTS
      value: "-Xmx512m"
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  - name: cosign
    image: gcr.io/projectsigstore/cosign:v2.4.0
    command: [sleep, infinity]
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
                    sh './mvnw -B -DskipTests package'
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
                archiveArtifacts artifacts: 'target/spring-petclinic-*.jar', fingerprint: true
                archiveArtifacts artifacts: 'target/bom.json, target/bom.xml', allowEmptyArchive: true
            }
        }

        stage('SLSA L1 Provenance') {
            steps {
                container('maven') {
                    script {
                        def buildTime = sh(script: 'date -u +"%Y-%m-%dT%H:%M:%SZ"', returnStdout: true).trim()
                        def jarFile   = 'target/spring-petclinic-4.0.0-SNAPSHOT.jar'
                        def jarSha256 = sh(
                            script: "sha256sum ${jarFile} | awk '{print \$1}'",
                            returnStdout: true
                        ).trim()

                        def provenance = """{
  "slsa_level": "1",
  "subject": {
    "name": "spring-petclinic-4.0.0-SNAPSHOT.jar",
    "digest": { "sha256": "${jarSha256}" }
  },
  "source": {
    "repository": "${env.GIT_REPO_URL}",
    "commit": "${env.GIT_COMMIT_SHA}",
    "branch": "${env.BRANCH_NAME ?: 'main'}"
  },
  "builder": {
    "id": "${env.JENKINS_URL}",
    "platform": "CloudBees CI on EKS"
  },
  "build": {
    "url": "${env.BUILD_URL}",
    "job": "${env.JOB_NAME}",
    "number": "${env.BUILD_NUMBER}",
    "tool": "maven:3.9-eclipse-temurin-17"
  },
  "metadata": {
    "built_at": "${buildTime}",
    "sbom": "target/bom.json"
  }
}"""
                        writeFile file: 'provenance-l1.json', text: provenance
                        echo "=== SLSA L1 Provenance ==="
                        sh 'cat provenance-l1.json'
                    }
                }
                archiveArtifacts artifacts: 'provenance-l1.json', fingerprint: true
            }
        }

       stage('Verify cosign') {
             steps {
                 container('cosign') {
                     sh 'cosign version'
                 }
    }
}

    }

    post {
        success { echo "Build succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
        failure { echo "Build failed: check the stage logs above" }
    }
}
