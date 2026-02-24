pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "ecommerce-mobile-store"
        DOCKER_REGISTRY = "docker.io"
        PROJECT_DIR = "/var/jenkins_home/workspace/ecommerce-build"
    }

    stages {
        stage('Clone Repository') {
            steps {
                echo 'Cloning from GitLab...'
                deleteDir()
                git(
                    url: '${GITLAB_REPO_URL}',
                    branch: '${BRANCH}',
                    credentialsId: 'gitlab-credentials'
                )
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                script {
                    sh '''
                        docker build \
                            -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                            -t ${DOCKER_IMAGE}:latest \
                            .
                    '''
                }
            }
        }

        stage('Run Tests') {
            steps {
                echo 'Running PHP syntax tests...'
                script {
                    sh '''
                        docker run --rm ${DOCKER_IMAGE}:latest php -l index.php
                        docker run --rm ${DOCKER_IMAGE}:latest php -l includes/db.php
                        docker run --rm ${DOCKER_IMAGE}:latest php -l includes/setup.php
                        docker run --rm ${DOCKER_IMAGE}:latest php -l auth/login.php
                        echo "✓ All PHP files passed syntax check"
                    '''
                }
            }
        }

        stage('Push to Docker Hub') {
            when {
                branch 'main'
            }
            steps {
                echo 'Pushing image to Docker Hub...'
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh '''
                            echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
                            docker tag ${DOCKER_IMAGE}:latest ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE}:${BUILD_NUMBER}
                            docker tag ${DOCKER_IMAGE}:latest ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE}:latest
                            docker push ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE}:${BUILD_NUMBER}
                            docker push ${DOCKER_REGISTRY}/${DOCKER_USERNAME}/${DOCKER_IMAGE}:latest
                            echo "✓ Image pushed to Docker Hub"
                        '''
                    }
                }
            }
        }

        stage('Deploy to Local AKS') {
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to AKS cluster...'
                script {
                    withCredentials([file(credentialsId: 'aks-kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            export KUBECONFIG=$KUBECONFIG_FILE
                            kubectl apply -f k8s-deployment.yml
                            kubectl rollout status deployment/ecommerce --timeout=5m
                            echo "✓ Deployment successful"
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo '✓ Pipeline completed successfully!'
            // Optional: Send notification
        }
        failure {
            echo '✗ Pipeline failed!'
            // Optional: Send notification
        }
    }
}
