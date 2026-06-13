pipeline {
    agent any

    environment {
        SONAR_HOME = tool 'sonarqube'
        IMAGE_NAME = "banking-app"
        ECR_REPO = "123456789.dkr.ecr.us-east-1.amazonaws.com/banking-app"
        TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/org/banking-app.git'
            }
        }

        stage('Gitleaks Scan') {
            steps {
                sh 'gitleaks detect --source . --exit-code 1'
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan .',
                odcInstallation: 'OWASP-DC'
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                docker build -t ${IMAGE_NAME}:${TAG} .
                """
            }
        }

        stage('Trivy Scan') {
            steps {
                sh """
                trivy image --exit-code 1 --severity HIGH,CRITICAL \
                ${IMAGE_NAME}:${TAG}
                """
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region us-east-1 | \
                docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

                docker tag ${IMAGE_NAME}:${TAG} ${ECR_REPO}:${TAG}
                docker push ${ECR_REPO}:${TAG}
                """
            }
        }

        stage('Helm Lint') {
            steps {
                sh 'helm lint helm-chart/'
            }
        }

        stage('Deploy Green Environment') {
            steps {
                sh """
                helm upgrade --install banking-green helm-chart/ \
                --set image.repository=${ECR_REPO} \
                --set image.tag=${TAG}
                """
            }
        }

        stage('Smoke Test') {
            steps {
                sh 'curl -f http://green-app/actuator/health'
            }
        }

        stage('Blue-Green Switch') {
            steps {
                sh 'kubectl apply -f service-green.yaml'
            }
        }

        stage('OWASP ZAP DAST') {
            steps {
                sh """
                docker run -t owasp/zap2docker-stable zap-baseline.py \
                -t http://prod-app-url
                """
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#devsecops',
                message: "SUCCESS: Build ${BUILD_NUMBER} deployed successfully"
            )
        }

        failure {
            slackSend(
                channel: '#devsecops',
                message: "FAILED: Build ${BUILD_NUMBER} failed"
            )
        }
    }
}
