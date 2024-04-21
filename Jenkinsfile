pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker')
    }

    stages {
        stage('Verify Branch') {
            steps {
                echo "$GIT_BRANCH"
            }
        }

        stage('Login to Dockerhub') {
            steps {
                script {
                    // Retrieve Dockerhub credentials from Jenkins
                    withCredentials([usernamePassword(
                        credentialsId: 'docker', 
                        passwordVariable: 'DOCKERHUB_CREDENTIALS_PSW', 
                        usernameVariable: 'DOCKERHUB_CREDENTIALS_USR'
                    )]) {
                        // Log in to Dockerhub
                        sh "docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}"
                    }
                }
            }
        }

        stage('AWS Credentials') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "aws",
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {
                    sh 'echo $AWS_ACCESS_KEY_ID'
                    sh 'echo $AWS_SECRET_ACCESS_KEY'
                }
            }
        }

        stage('Pulling base image from Dockerhub') {
            steps {
                sh 'docker pull 5ggraduationproject/pcf-base'
            }
        }

        stage('docker build') {
            steps {
                sh(script: """
                    docker images -a
                    docker build -t asmaamaged/pcf:latest .
                    docker images -a
                """)
            }
        }

        stage('Scan Image for Common Vulnerabilities and Exposures') {
            steps {
                sh 'trivy image asmaamaged/5g pcf --output trivy-report.json'
            }
        }

        stage('Pushing to Dockerhub') {
            steps {
                sh 'docker push asmaamaged/5g-pcf:latest'
            }
        }

        stage('Build and Package Helm Chart') {
            steps {
                sh 'helm package ./helm-pcf/'
            }
        }

        stage('Configure Kubernetes Context') {
            steps {
                sh 'aws eks --region us-east-1 update-kubeconfig --name 5G-Core-Net'
            }
        }

        stage('Deploy Helm Chart on EKS') {
            steps {
                sh 'helm upgrade --install pcf ./helm-upf/'
            }
        }
    }

    post {
        always {
            // Archiving Test Result
            archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
            sh 'docker rmi asmaamaged/5g-pcf:latest'
        }
    }
}