pipeline {
  agent any

  tools {
    jdk 'jdk17'
    nodejs 'node23'
  }

  environment {
    DOCKER_HUB_USER = "balu361988"
    IMAGE_NAME = "small-project"
    IMAGE_TAG = "v${BUILD_NUMBER}"
    FULL_IMAGE_NAME = "${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
    SCANNER_HOME = tool 'sonar-scanner'
    KUBECONFIG = '/var/lib/jenkins/.kube/config'
  }

  stages {
    stage('Clean Workspace') {
      steps {
        cleanWs()
      }
    }

    stage('Checkout Code') {
      steps {
        checkout scm
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonar-server') {
          sh '''
            ${SCANNER_HOME}/bin/sonar-scanner \
              -Dsonar.projectKey=small-project \
              -Dsonar.projectName=small-project \
              -Dsonar.sources=.
          '''
        }
      }
    }

    stage('Quality Gate') {
      steps {
        script {
          waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
        }
      }
    }

    stage('Install NPM Dependencies') {
      steps {
        sh 'npm install'
      }
    }

    stage('OWASP Dependency Check') {
      steps {
        dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --out .', odcInstallation: 'DP-Check'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
      }
    }

    stage('Trivy FS Scan') {
      steps {
        sh "trivy fs --format table -o trivy-fs-report.html ."
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          dockerImage = docker.build("${FULL_IMAGE_NAME}", "--no-cache .")
        }
      }
    }

    stage('Push Docker Image') {
      steps {
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
            dockerImage.push()
          }
        }
      }
    }

    stage('Trivy Image Scan') {
      steps {
        sh "trivy image --format table -o trivy-image-report.html ${FULL_IMAGE_NAME}"
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        script {
          sh """
            sed -i 's|image: .*|image: ${FULL_IMAGE_NAME}|' k8s-deployment.yaml
            kubectl apply -f k8s-deployment.yaml --validate=false
            kubectl rollout status deployment/small-project
          """
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'trivy-*.html', allowEmptyArchive: true
      emailext(
        attachLog: true,
        subject: "'${currentBuild.result}' - ${env.IMAGE_TAG}",
        body: """
          <html>
          <body>
            <p><strong>Job:</strong> ${env.JOB_NAME}</p>
            <p><strong>Build:</strong> ${env.BUILD_NUMBER}</p>
            <p><strong>Image:</strong> ${env.FULL_IMAGE_NAME}</p>
            <p><strong>URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
          </body>
          </html>
        """,
        to: 'face361988@gmail.com',
        mimeType: 'text/html',
        attachmentsPattern: 'trivy-image-report.html'
      )
    }
  }
}

