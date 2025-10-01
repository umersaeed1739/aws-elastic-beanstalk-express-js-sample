pipeline {
    agent any

    environment {
        IMAGE_NAME = 'umersaeed732/nodejs-sample-app'
    }

    stages {
        stage('Prepare Jenkins environment') {
            steps {
                script {
                    def dockerExists = sh(script: 'which docker || echo "notfound"', returnStdout: true).trim()
                    if (dockerExists == 'notfound') {
                        echo 'Docker CLI not found, installing...'
                        sh '''
                            apt-get update
                            apt-get install -y curl
                            curl -fsSL https://get.docker.com -o get-docker.sh
                            sh get-docker.sh
                        '''
                    } else {
                        echo "Docker CLI found at ${dockerExists}"
                    }
                }
            }
        }

        stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Confirm Docker Root Dir') {
            steps {
                sh 'echo Docker Root Dir:'
                sh 'docker info | grep "Docker Root Dir"'
            }
        }

        stage('Verify Workspace Files') {
            steps {
                echo 'Listing files in workspace:'
                sh 'ls -la $WORKSPACE'
                echo 'Showing package.json content:'
                sh 'cat $WORKSPACE/package.json'
            }
        }

        stage('Verify Docker Mount') {
            steps {
                echo 'Listing files inside Docker container at /app:'
                sh '''
                    docker run --rm -v $WORKSPACE:/app -w /app node:16 ls -la /app
                    docker run --rm -v $WORKSPACE:/app -w /app node:16 cat package.json
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'docker run --rm -v $WORKSPACE:/app -w /app node:16 npm install'
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    docker run --rm -v /tmp/app:/app -w /app node:16 npm test
                '''
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                    docker run --rm -v /tmp/app:/app -w /app node:16 /bin/sh -c "
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

