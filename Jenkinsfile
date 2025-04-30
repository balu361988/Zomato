pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'  // ✅ match the exact configured name
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                git 'https://github.com/balu361988/Zomato.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=zomato \
                        -Dsonar.projectKey=zomato
                    '''
                }
            }
        }

        stage('Code Quality Gate') {
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

        stage('File system scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t zomato .'
            }
        }

        stage('Tag & Push to DockerHub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh 'docker tag zomato balu361988/zomato:latest'
                        sh 'docker push balu361988/zomato:latest'
                    }
                }
            }
        }

        stage('Docker Image scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html balu361988/zomato:latest"
            }
        }

        stage('Docker Scout Scan') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                            if docker scout version > /dev/null 2>&1; then
                                docker scout quickview balu361988/zomato:latest
                                docker scout cves balu361988/zomato:latest
                                docker scout recommendations balu361988/zomato:latest
                            else
                                echo "Docker Scout not installed. Skipping scout scans."
                            fi
                        '''
                    }
                }
            }
        }

            stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Deploying to Kubernetes with image: ${DOCKER_IMAGE}"
                    sh """
                        sed -i 's|image: .*|image: ${DOCKER_IMAGE}|' deployment.yaml
                        kubectl apply -f deployment.yaml --validate=false
                        kubectl rollout status deployment/zomato
                    """
                }
            }
        }
    } // ✅ Closing the 'stages' block

    post {
        always {
            emailext(
                attachLog: true,
                subject: "'${currentBuild.result}'",
                body: """
                    <html>
                    <body>
                        <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                            <p style="color: white; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                        </div>
                        <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                            <p style="color: white; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                        </div>
                        <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px;">
                            <p style="color: white; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                        </div>
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

