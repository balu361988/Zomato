pipeline {
  agent any

  environment {
    DOCKER_HUB_USER = "balu361988"
    IMAGE_NAME = "zomato"                       // 🔁 Changed from 'small-project'
    IMAGE_TAG = "v${BUILD_NUMBER}"
    FULL_IMAGE_NAME = "${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
  }

  stages {
    stage('Checkout Code') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          dockerImage = docker.build("${FULL_IMAGE_NAME}", "--no-cache .")
        }
      }
    }

    stage('Push Docker Image to DockerHub') {
      steps {
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
            dockerImage.push()
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      environment {
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
      }
      steps {
        script {
          echo "Deploying ${FULL_IMAGE_NAME} to Kubernetes..."
          sh """
            sed -i 's|image: .*|image: ${FULL_IMAGE_NAME}|' k8s-deployment.yaml
            kubectl apply -f k8s-deployment.yaml --validate=false
            kubectl rollout status deployment/zomato
          """
        }
      }
    }
  }

  post {
    success {
      echo "✅ Zomato deployed successfully!"
    }
    failure {
      echo "❌ Zomato deployment failed."
    }
  }
}

