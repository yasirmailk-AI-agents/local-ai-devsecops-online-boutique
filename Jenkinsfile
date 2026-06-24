pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 45, unit: 'MINUTES')
    }

    environment {
        REGISTRY           = 'localhost:5001'
        IMAGE_NAME         = 'frontend'
        IMAGE_TAG          = "${BUILD_NUMBER}"
        SONAR_PROJECT_KEY  = 'online-boutique'
        SONAR_PROJECT_NAME = 'Online Boutique'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Repository Structure') {
            steps {
                sh '''
                    set -eux
                    pwd
                    ls -la
                    ls -la src/frontend
                    test -d src/frontend
                    test -f src/frontend/Dockerfile
                    test -f src/frontend/go.mod
                '''
            }
        }

        stage('Secret Scan - Gitleaks') {
            steps {
                sh '''
                    set -eux
                    docker run --rm \
                      -v "${WORKSPACE}:/repo" \
                      zricethezav/gitleaks:latest \
                      detect \
                      --source=/repo \
                      --redact \
                      --exit-code=1
                '''
            }
        }

        stage('Filesystem Scan - Trivy') {
            steps {
                sh '''
                    set -eux
                    docker run --rm \
                      -v "${WORKSPACE}:/workspace:ro" \
                      -v trivy-cache:/root/.cache/trivy \
                      aquasec/trivy:latest \
                      fs \
                      --scanners vuln,misconfig \
                      --severity CRITICAL \
                      --exit-code 1 \
                      --no-progress \
                      /workspace
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            set -eux
                            "${scannerHome}/bin/sonar-scanner" \
                              -Dsonar.projectKey="${SONAR_PROJECT_KEY}" \
                              -Dsonar.projectName="${SONAR_PROJECT_NAME}" \
                              -Dsonar.sources=src/frontend \
                              -Dsonar.sourceEncoding=UTF-8 \
                              -Dsonar.scm.provider=git \
                              -Dsonar.exclusions="**/node_modules/**,**/vendor/**,**/generated/**,**/dist/**,**/build/**"
                        """
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                    set -eux
                    docker build \
                      --pull \
                      -t "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}" \
                      -t "${REGISTRY}/${IMAGE_NAME}:latest" \
                      src/frontend
                '''
            }
        }

        stage('Image Scan - Trivy') {
            steps {
                sh '''
                    set -eux
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      -v trivy-cache:/root/.cache/trivy \
                      aquasec/trivy:latest \
                      image \
                      --scanners vuln \
                      --severity CRITICAL \
                      --exit-code 1 \
                      --no-progress \
                      "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                '''
            }
        }

        stage('Push Local Registry') {
            steps {
                sh '''
                    set -eux
                    docker push "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                    docker push "${REGISTRY}/${IMAGE_NAME}:latest"
                '''
            }
        }

        stage('Verify Registry Image') {
            steps {
                sh '''
                    set -eux
                    docker pull "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                    docker image inspect "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline successful"
            echo "Image pushed: ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
        }

        failure {
            echo "Pipeline failed. Failed stage ke logs check karein."
        }

        always {
            echo "Pipeline completed with status: ${currentBuild.currentResult}"
        }
    }
}
