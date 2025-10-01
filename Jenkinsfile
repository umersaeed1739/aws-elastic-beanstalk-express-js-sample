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
		    # Define the staging directory variable locally for shell script use
		    APP_STAGING_DIR=/var/jenkins_home/workspace/20294728_Project2_pipeline@2/temp_app 
		    
		    rm -rf \$APP_STAGING_DIR
		    mkdir -p \$APP_STAGING_DIR
		    
		    # Copy all necessary files and set permissions/ownership
		    cp -r CODE_OF_CONDUCT.md CONTRIBUTING.md Jenkinsfile LICENSE README.md app.js docker_build package-lock.json package.json \$APP_STAGING_DIR/
		    chown -R 1000:1000 \$APP_STAGING_DIR
		    rm -rf \$APP_STAGING_DIR/node_modules
		"""
		// NEW: Use a separate 'sh' step for the Dockerfile creation
		sh """
		    APP_STAGING_DIR=/var/jenkins_home/workspace/20294728_Project2_pipeline@2/temp_app 
		    
		    cat <<EOF > \${APP_STAGING_DIR}/Dockerfile.test
		    FROM node:16
		    WORKDIR /app
		    COPY . /app
		    RUN npm install
		    EOF
		    
		    # Optional: Check the contents of the generated file
		    echo "--- Dockerfile.test contents ---"
		    cat \${APP_STAGING_DIR}/Dockerfile.test
		    echo "------------------------------"
		"""
	    }
	}


// ----------------------------------------------------------------------

	stage('Install Dependencies') {
	    steps {
		echo 'Building test image with dependencies installed...'
		// Build a temporary image based on the staging directory and the Dockerfile.test
		sh "docker build -t node-app-test:latest -f \${APP_STAGING_DIR}/Dockerfile.test \${APP_STAGING_DIR}"
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
