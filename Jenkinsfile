pipeline {
    agent any

    environment {
        IMAGE_NAME = "ahmedali772/java-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/engrahmadaliq-sudo/java-app.git'
            }
        }


        stage('Build & Unit Test') {
            steps {
                sh 'mvn clean package'
            }

            post {
                always {
                    script {
                        junit allowEmptyResults: true,
                        testResults: 'target/surefire-reports/*.xml'
                    }
                }
            }
        }


        stage('Static Code Analysis') {
            steps {
                echo 'SonarQube analysis skipped'
            }
        }


        stage('Docker Build') {
            steps {
                sh """
                docker build \
                -t ${IMAGE_NAME}:${IMAGE_TAG} \
                -t ${IMAGE_NAME}:latest .
                """
            }
        }


        stage('Docker Push') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh """
                    echo \$DOCKER_PASS | docker login \
                    -u \$DOCKER_USER \
                    --password-stdin

                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest

                    docker logout
                    """
                }
            }
        }


        stage('Deploy to Kubernetes') {

            steps {

                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh """
                    echo "Deploying to Kubernetes..."

                    export KUBECONFIG=\$KUBECONFIG

                    kubectl get nodes

                    kubectl set image \
                    deployment/java-app-deployment \
                    java-app=${IMAGE_NAME}:${IMAGE_TAG}

                    kubectl rollout status \
                    deployment/java-app-deployment \
                    --timeout=300s
                    """
                }
            }
        }
    }


    post {

        success {
            echo "Deployment successful: ${IMAGE_NAME}:${IMAGE_TAG}"
        }


        failure {
            echo "Pipeline failed"
        }


        always {
            echo "Cleaning workspace..."
            cleanWs()
        }
    }
}
