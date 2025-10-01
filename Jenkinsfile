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
		    # First shell block: copy files, set permissions (no heredoc here)
		    APP_STAGING_DIR=/var/jenkins_home/workspace/20294728_Project2_pipeline@2/temp_app 
		    
		    rm -rf \$APP_STAGING_DIR
		    mkdir -p \$APP_STAGING_DIR
		    
		    # Copy files and set permissions/ownership
		    cp -r CODE_OF_CONDUCT.md CONTRIBUTING.md Jenkinsfile LICENSE README.md app.js docker_build package-lock.json package.json \$APP_STAGING_DIR/
		    chown -R 1000:1000 \$APP_STAGING_DIR
		    rm -rf \$APP_STAGING_DIR/node_modules
		"""
		// SECOND SHELL BLOCK: Use dashed heredoc and ensure NO INDENTATION for the content and closing EOF
		sh """
	APP_STAGING_DIR=/var/jenkins_home/workspace/20294728_Project2_pipeline@2/temp_app 

	cat <<-EOF > \${APP_STAGING_DIR}/Dockerfile.test
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
		// FIX: Use the 'node-app-test:latest' image, which contains the code and dependencies.
		sh 'docker run --rm node-app-test:latest npm test'
	    }
	}

	stage('Security Scan') {
	    steps {
		// FIX: Run snyk inside the 'node-app-test:latest' image
		sh '''
		    docker run --rm node-app-test:latest /bin/sh -c "
		    # npm is available since dependencies were installed in the build step
		    npm install -g snyk && snyk test || exit 1
		    "
		'''
	    }
	}

	stage('Build Docker Image') {
	    steps {
		# Assuming the app's *production* Dockerfile is located in the staging directory
		sh "docker build -t \$IMAGE_NAME \${APP_STAGING_DIR}"
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
