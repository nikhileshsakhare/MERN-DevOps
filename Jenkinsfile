pipeline {
    agent any

    tools {
        nodejs 'NodeJS-24.14.1'
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

        stage('Lint') {
            parallel {
                stage('Lint frontend') {
                    steps {
                        dir('frontend') { sh 'npm run lint' }
                    }
                }
                stage('Lint backend') {
                    steps {
                        dir('backend') { sh 'npm run lint' }
                    }
                }
            }
        }

        stage('Test') {
            steps {
                dir('backend') {
                    sh 'npm test'
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
                    docker build -t $IMAGE_NAME-backend:$BUILD_NUMBER ./backend
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
