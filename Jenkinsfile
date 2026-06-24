stage('SonarQube Analysis') {
    steps {
        script {
            def scannerHome = tool 'SonarScanner'

            withSonarQubeEnv('SonarQube') {
                sh """
                    ${scannerHome}/bin/sonar-scanner \
                      -Dsonar.projectKey=online-boutique \
                      -Dsonar.projectName="Online Boutique" \
                      -Dsonar.sources=src \
                      -Dsonar.sourceEncoding=UTF-8 \
                      -Dsonar.exclusions=**/node_modules/**,**/vendor/**,**/generated/**,**/dist/**,**/build/**
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
stage('Build Frontend Image')pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  environment {
    REGISTRY = 'localhost:5001'
    IMAGE_NAME = 'online-boutique-frontend'
    IMAGE_TAG = "${BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD'
      }
    }

    stage('Repository Structure') {
      steps {
        sh '''
          test -d src/frontend
          test -f src/frontend/Dockerfile
          test -f skaffold.yaml
        '''
      }
    }

    stage('Secret Scan - Gitleaks') {
      steps {
        sh '''
          docker run --rm \
            --volumes-from devops-jenkins \
            -w "$WORKSPACE" \
            ghcr.io/gitleaks/gitleaks:latest \
            detect --source=. --no-banner --redact
        '''
      }
    }

    stage('Filesystem Scan - Trivy') {
      steps {
        sh '''
          docker run --rm \
            --volumes-from devops-jenkins \
            -w "$WORKSPACE" \
            aquasec/trivy:latest \
            fs --scanners vuln,secret,misconfig \
            --severity HIGH,CRITICAL \
            --exit-code 0 .
        '''
      }
    }

    stage('Build Frontend Image') {
      steps {
        sh '''
          docker build \
            -t ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
            src/frontend
        '''
      }
    }

    stage('Image Scan - Trivy') {
      steps {
        sh '''
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest \
            image --severity CRITICAL --exit-code 1 \
            ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
        '''
      }
    }

    stage('Push Local Registry') {
      steps {
        sh 'docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}'
      }
    }
  }

  post {
    always {
      sh 'docker image ls ${REGISTRY}/${IMAGE_NAME} || true'
    }
    success {
      echo "Image ready: ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
      echo 'Next GitOps step: update the frontend image tag in the GitOps repository and let Argo CD sync it.'
    }
  }
}
