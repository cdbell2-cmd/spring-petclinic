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

        stage('Setup') {
            steps {
                container('maven') {
                    sh '''
                        COSIGN_VERSION=v2.4.0
                        curl -sSfLo /usr/local/bin/cosign \
                            "https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64"
                        chmod +x /usr/local/bin/cosign
                        cosign version
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    sh './mvnw -B -DskipTests package cyclonedx:makeAggregateBom'
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

        stage('SLSA L2 Provenance') {
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
  "buildDefinition": {
    "buildType": "https://cloudbees.com/jenkinsfile/v1",
    "externalParameters": {
      "repository": "${env.GIT_REPO_URL}",
      "ref": "${env.GIT_COMMIT_SHA}",
      "branch": "${env.BRANCH_NAME ?: 'main'}"
    },
    "internalParameters": {
      "buildUrl": "${env.BUILD_URL}",
      "controllerId": "${env.JENKINS_URL}",
      "jobName": "${env.JOB_NAME}",
      "buildNumber": "${env.BUILD_NUMBER}",
      "buildTool": "maven:3.9-eclipse-temurin-17"
    }
  },
  "runDetails": {
    "builder": {
      "id": "https://cloudbees-ci.proofpoint.com"
    },
    "metadata": {
      "invocationId": "${env.BUILD_URL}",
      "startedOn": "${buildTime}"
    }
  },
  "subject": [
    {
      "name": "spring-petclinic-4.0.0-SNAPSHOT.jar",
      "digest": { "sha256": "${jarSha256}" }
    }
  ]
}"""
                        writeFile file: 'provenance-l2.json', text: provenance
                        sh 'cat provenance-l2.json'
                    }

                    withCredentials([file(credentialsId: 'cosign-private-key', variable: 'COSIGN_KEY_PATH')]) {
                        sh '''
                            export COSIGN_PASSWORD=""
                            JAR="target/spring-petclinic-4.0.0-SNAPSHOT.jar"

                            cosign sign-blob \
                                --key "${COSIGN_KEY_PATH}" \
                                --output-signature "${JAR}.sig" \
                                --tlog-upload=false \
                                "${JAR}"

                            cosign sign-blob \
                                --key "${COSIGN_KEY_PATH}" \
                                --output-signature "provenance-l2.json.sig" \
                                --tlog-upload=false \
                                "provenance-l2.json"

                            echo "Signed artifacts:"
                            ls -lh "${JAR}.sig" "provenance-l2.json.sig"
                        '''
                    }
                }

                archiveArtifacts artifacts: 'provenance-l2.json, provenance-l2.json.sig, target/spring-petclinic-4.0.0-SNAPSHOT.jar.sig'
            }
        }
    }

    post {
        success { echo "SLSA L2 build complete: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
        failure { echo "Build failed: check the stage logs above" }
    }
}
