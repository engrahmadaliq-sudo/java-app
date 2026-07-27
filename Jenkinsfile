pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker-hub-creds')   // Jenkins credential ID
        KUBECONFIG_CREDENTIALS = credentials('kubeconfig')        // Jenkins credential ID (Secret file)
        IMAGE_NAME = "yourdockerhubuser/java-app"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/yourusername/java-app.git'
            }
        }

        stage('Build & Unit Test') {
            steps {
                sh 'mvn clean package'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Static Code Analysis') {
            steps {
                echo 'Running static analysis (SonarQube optional)...'
                // sh 'mvn sonar:sonar -Dsonar.projectKey=java-app -Dsonar.host.url=$SONAR_URL -Dsonar.login=$SONAR_TOKEN'
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Docker Push') {
            steps {
                sh "echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin"
                sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    export KUBECONFIG=${KUBECONFIG_CREDENTIALS}
                    kubectl set image deployment/java-app-deployment java-app=${IMAGE_NAME}:${IMAGE_TAG} --record=true
                    kubectl rollout status deployment/java-app-deployment --timeout=120s
                """
            }
        }
    }

    post {
        success {
            echo "Deployment successful: ${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline failed. Rolling back to previous revision..."
            sh "export KUBECONFIG=${KUBECONFIG_CREDENTIALS} && kubectl rollout undo deployment/java-app-deployment || true"
        }
        always {
            sh 'docker logout || true'
        }
    }
}
