pipeline {
    agent any

    environment {
        // Docker Hub credentials ID stored in Jenkins
        DOCKERHUB_CREDENTIALS ='cybr-3120'
        IMAGE_NAME ='amalan06/amalangametest123'
    }

    stages {

        stage('Cloning Git') {
            steps {
                checkout scm
            }
        }

        stage('SAST') {
            steps {
                sh 'echo Running SAST scan...'
            }
        }

      stage('BUILD-AND-TAG') {
            agent {
                label 'CYBR3120-01-app-server'
            }
            steps {
                script {
                    // Build Docker image using Jenkins Docker Pipeline API
                    echo "Building Docker image ${IMAGE_NAME}..."
                    app = docker.build("${IMAGE_NAME}")
                    app.tag("latest")
                }
            }
        }


        stage('POST-TO-DOCKERHUB') {    
            agent {
                label 'CYBR3120-01-app-server'
            }
            steps {
                script {
                    echo "Pushing image ${IMAGE_NAME}:latest to Docker Hub..."
                    docker.withRegistry('https://registry.hub.docker.com', "${DOCKERHUB_CREDENTIALS}") {
                        app.push("latest")
                    }
                }
            }
        }

        stage('SECURITY-IMAGE-SCANNER') {
            steps {
                sh 'echo Scanning Docker image for vulnerabilities...'
            }
        }

        stage('Pull-image-server') {
            steps {
                sh 'echo Pulling image on server...'
            }
        }

        stage('DAST') {
            steps {
                sh 'echo Performing DAST scan...'
            }
        }

        stage('DEPLOYMENT') {    
            agent {
                label 'CYBR3120-01-app-server'
            }
            steps {
                echo 'Starting deployment using docker-compose...'
                script {
                    dir("${WORKSPACE}") {
                        sh '''
                            docker-compose down
                            docker-compose up -d
                            docker ps
                        '''
                    }
                }
                echo 'Deployment completed successfully!'
            }
        }
    }  
}  