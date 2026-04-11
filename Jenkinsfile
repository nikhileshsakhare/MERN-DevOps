pipeline {
    agent any

    tools {
        nodejs 'NodeJS 24.14.1'
    }

    environment {
        MONGO_URI      = credentials('mongo-uri')
        DOCKERHUB_CRED = credentials('dockerhub-credentials')
        IMAGE_NAME     = 'nikhilesh2701/mern-app'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Cloning repository...'
                checkout scm
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('frontend deps') {
                    steps {
                        dir('frontend') {
                            sh 'npm ci'
                        }
                    }
                }
                stage('backend deps') {
                    steps {
                        dir('backend') {
                            sh 'npm ci'
                        }
                    }
                }
            }
        }

        stage('Lint frontend') {
            steps {
                dir('frontend') {
                    sh 'npm run lint -- --max-warnings 100'
                }
            }
        }

        stage('Build React App') {
            steps {
                dir('frontend') {
                    sh 'npm run build'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -t $IMAGE_NAME-frontend:$BUILD_NUMBER ./frontend
                    docker build -t $IMAGE_NAME-backend:$BUILD_NUMBER -f backend/Dockerfile .
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh '''
                    echo $DOCKERHUB_CRED_PSW | docker login -u $DOCKERHUB_CRED_USR --password-stdin
                    docker push $IMAGE_NAME-frontend:$BUILD_NUMBER
                    docker push $IMAGE_NAME-backend:$BUILD_NUMBER
                '''
            }
        }

        stage('Setup Kubeconfig') {
            steps {
                sh '''
                    kubectl config view --minify --flatten > /home/ec2-user/.kube/config-flat
                    cp /home/ec2-user/.kube/config-flat /var/jenkins_home/.kube/config
                    chown -R jenkins:jenkins /var/jenkins_home/.kube
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    sed -i "s|$IMAGE_NAME-frontend:latest|$IMAGE_NAME-frontend:$BUILD_NUMBER|g" K8s/client/deployment.yaml
                    sed -i "s|$IMAGE_NAME-backend:latest|$IMAGE_NAME-backend:$BUILD_NUMBER|g" K8s/server/deployment.yaml

                    kubectl apply -f K8s/namespace.yaml
                    kubectl apply -f K8s/secrets.yaml
                    kubectl apply -f K8s/mongo/
                    kubectl apply -f K8s/server/
                    kubectl apply -f K8s/client/

                    kubectl rollout status deployment/server -n mern-app
                    kubectl rollout status deployment/client -n mern-app
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs above.'
        }
        always {
            sh 'docker logout'
        }
    }
}
