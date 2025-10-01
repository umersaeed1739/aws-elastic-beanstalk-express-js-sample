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

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Confirm Docker Root Dir') {
            steps {
                sh '''
                    echo "Docker Root Dir:"
                    docker info | grep "Docker Root Dir"
                '''
            }
        }

        stage('Prepare Workspace for Docker') {
            steps {
                sh '''
                    rm -rf /tmp/app || true
                    mkdir -p /tmp/app
                    cd $WORKSPACE
                    tar cf - CODE_OF_CONDUCT.md CONTRIBUTING.md Jenkinsfile LICENSE README.md app.js package-lock.json package.json | (cd /tmp/app && tar xf -)
                    ls -la /tmp/app
                '''
            }
        }

        stage('Verify Docker Mount') {
            steps {
                sh '''
                    docker run --rm -v /tmp/app:/app -w /app node:16 ls -la /app
                    docker run --rm -v /tmp/app:/app -w /app node:16 cat package.json || echo "package.json not found"
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    docker run --rm -v /tmp/app:/app -w /app node:16 npm install
                '''
            }
        }

        // ... remaining stages as before

    }

    post {
        always {
            archiveArtifacts artifacts: '**/*.log', allowEmptyArchive: true
        }
    }
}

