pipeline {
    agent any

    environment {
        IMAGE_NAME = 'umersaeed732/nodejs-sample-app'
        // NEW: Define a reliable staging directory within the Jenkins workspace
        APP_STAGING_DIR = "${env.WORKSPACE}/temp_app" 
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
                sh 'echo Docker Root Dir:'
                sh 'docker info | grep "Docker Root Dir"'
            }
        }
        
// ----------------------------------------------------------------------
        
        stage('Prepare Workspace for Docker') {
            steps {
                sh """
                    # FIX: Use the new APP_STAGING_DIR variable for a reliable mount point
                    rm -rf \$APP_STAGING_DIR
                    mkdir -p \$APP_STAGING_DIR
                    # Copy all contents from the current workspace into the staging directory
                    cp -r * \$APP_STAGING_DIR/
                    ls -la \$APP_STAGING_DIR/
                """
            }
        }

// ----------------------------------------------------------------------
        
        stage('Verify Docker Mount') {
            steps {
                echo 'Listing files inside Docker container at /app:'
                sh """
                    # FIX: Use the new APP_STAGING_DIR variable in the volume mount
                    docker run --rm -v \${APP_STAGING_DIR}:/app -w /app node:16 ls -la /app
                    docker run --rm -v \${APP_STAGING_DIR}:/app -w /app node:16 cat package.json
                """
            }
        }

// ----------------------------------------------------------------------

        stage('Install Dependencies') {
            steps {
                // FIX: Replace /tmp/app with APP_STAGING_DIR
                sh 'docker run --rm -v ${APP_STAGING_DIR}:/app -w /app node:16 npm install'
            }
        }

        stage('Run Tests') {
            steps {
                // FIX: Replace /tmp/app with APP_STAGING_DIR
                sh 'docker run --rm -v ${APP_STAGING_DIR}:/app -w /app node:16 npm test'
            }
        }

        stage('Security Scan') {
            steps {
                // FIX: Replace /tmp/app with APP_STAGING_DIR
                sh '''
                    docker run --rm -v ${APP_STAGING_DIR}:/app -w /app node:16 /bin/sh -c "
                    npm install -g snyk && snyk test || exit 1
                    "
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                // FIX: Replace /tmp/app with APP_STAGING_DIR
                sh 'docker build -t $IMAGE_NAME ${APP_STAGING_DIR}'
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
