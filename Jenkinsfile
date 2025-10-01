pipeline {
    agent any

    environment {
        IMAGE_NAME = 'umersaeed732/nodejs-sample-app'
    }

    stages {
        stage('Prepare Jenkins environment') {
            steps {
                script {
                    // Check if docker CLI is installed, install if missing
                    def dockerExists = sh(script: 'which docker || echo "notfound"', returnStdout: true).trim()
                    if (dockerExists == 'notfound') {
                        echo 'Docker CLI not found, installing...'
                        // Assuming Debian/Ubuntu based Jenkins image
                        sh '''
                            apt-get update && \
                            apt-get install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common && \
                            curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add - && \
                            add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable" && \
                            apt-get update && \
                            apt-get install -y docker-ce-cli
                        '''
                    } else {
                        echo "Docker CLI found at ${dockerExists}"
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'docker run --rm -v $PWD:/app -w /app node:16 npm install'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'docker run --rm -v $PWD:/app -w /app node:16 npm test'
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                    docker run --rm -v $PWD:/app -w /app node:16 /bin/sh -c "
                    npm install -g snyk && snyk test || exit 1
                    "
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $IMAGE_NAME ."
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE_NAME
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '**/*.log', allowEmptyArchive: true
        }
    }
}

