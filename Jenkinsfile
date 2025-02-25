pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node19'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Checkout from Git') {
            steps {
                git branch: 'legacy', url: 'https://github.com/Routparesh/chatbot-ui'
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=Chatbot \
                    -Dsonar.projectKey=Chatbot
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
        
        stage('OWASP Dependency Check') {
            steps {
                // Download and run OWASP Dependency Check
                sh '''
                curl -L https://github.com/jeremylong/DependencyCheck/releases/download/v6.4.0/dependency-check-6.4.0-release.zip -o dependency-check.zip
                unzip -o dependency-check.zip
                chmod +x dependency-check/bin/dependency-check.sh
                '''
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    sh './dependency-check/bin/dependency-check.sh --scan ./ --format XML --disableYarnAudit --disableNodeAudit'
                }
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs . > trivyfs.json'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build -t chatbot .'
                        sh 'docker tag chatbot routparesh/chatbot:latest'
                        sh 'docker push routparesh/chatbot:latest'
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image routparesh/chatbot:latest > trivy.json'
            }
        }

        stage('Remove Existing Container') {
            steps {
                sh 'docker stop chatbot || true'
                sh 'docker rm chatbot || true'
            }
        }

        stage('Deploy to Container') {
            steps {
                sh 'docker run -d --name chatbot -p 3000:3000 routparesh/chatbot:latest'
            }
        }

        stage('Deploy to kubernets'){
            steps{
                script{
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                      sh 'kubectl apply -f k8s/chatbot-ui.yaml'
                  }
                }
            }
        }
    }
}
