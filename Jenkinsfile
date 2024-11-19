pipeline {
    agent any
    environment {
        SOURCE_REPO_URL = 'https://github.com/poppyszn/ibm-apps.git'
        BAR_REPO_URL = 'https://github.com/poppyszn/ibm-bar-files.git'
        CICD_REPO_URL = 'https://github.com/poppyszn/ibm_ace_cicd.git'
        DOCKER_IMAGE = 'preciousokpor/ibm-ace-builder:12.0.12.0'
        ACE_WORKSPACE_DIR = "${WORKSPACE}/workspace"
        LICENSE = 'accept'  // Required for IBM ACE
        DISPLAY = ':99'     // Required for Xvfb
        DOCKERHUB_REPO = 'preciousokpor/ibm-barfiles-image'
    }

    stages {
        stage('Checkout Source Repository') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    script {
                        def sourceRepoUrl = SOURCE_REPO_URL.replace("https://", "https://${GITHUB_TOKEN}@")

                        sh "git config --global --add safe.directory ${env.WORKSPACE}/source-repo"

                        if (fileExists('source-repo')) {
                            echo "Source repository already exists. Checking for updates..."
                            dir('source-repo') {
                                sh 'git fetch origin'
                                sh 'git reset --hard origin/main'
                            }
                        } else {
                            echo "Source repository does not exist. Cloning the repository..."
                            sh "git clone ${sourceRepoUrl} source-repo"
                        }
                    }
                }
            }
        }

        stage('Determine Application Names') {
            steps {
                script {
                    def appNames = sh(
                        script: "find source-repo -maxdepth 1 -type d -not -path 'source-repo' -not -name '.git' -exec basename {} \\;",
                        returnStdout: true
                    ).trim().split('\n')

                    env.APP_NAMES = appNames.join(' ')
                    echo "Applications to add to BAR file: ${env.APP_NAMES}"
                }
            }
        }

        stage('Build BAR File') {
            steps {
                script {
                    def barFilePath = "${WORKSPACE}/output.bar"
                    docker.image(DOCKER_IMAGE).inside('--entrypoint=""') {
                        sh """
                        echo "Starting BAR file creation process..."

                        bash -c '
                        # Start Xvfb for display requirements
                        Xvfb :99 -screen 0 1024x768x16 &

                        # Set environment variables
                        export DISPLAY=:99
                        export LICENSE=accept

                        # Navigate to source-repo
                        cd /var/jenkins_home/workspace/ibm-ace/source-repo

                        # Run mqsicreatebar with all applications at once
                        echo "Adding applications to BAR file: ${env.APP_NAMES}"
                        mqsicreatebar -data . -b ${barFilePath} -a ${env.APP_NAMES} -cleanBuild -trace

                        # Verify BAR file
                        ls -lha ${barFilePath}
                        '
                        """
                    }
                }
            }
        }

        stage('Update BAR Repository') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    script {
                        def barRepoUrl = BAR_REPO_URL.replace("https://", "https://${GITHUB_TOKEN}@")

                        // Checkout BAR repository
                        if (fileExists('bar-repo')) {
                            echo "BAR repository already exists. Pulling the latest changes..."
                            dir('bar-repo') {
                                sh """
                                git reset --hard
                                git clean -fd
                                git pull origin main
                                """
                            }
                        } else {
                            echo "Cloning the BAR repository..."
                            sh "git clone ${barRepoUrl} bar-repo"
                        }

                        // Copy BAR file and push to repository
                        dir('bar-repo') {
                            sh """
                            echo "Copying BAR file to the BAR repository..."
                            cp ${WORKSPACE}/output.bar .

                            echo "Committing the BAR file to the repository..."
                            git config user.name "jenkins-bot"
                            git config user.email "jenkins@example.com"
                            git add output.bar
                            git commit -m "Add updated BAR file: ${env.IMAGE_TAG}"
                            git push origin main
                            """
                        }
                    }
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    def currentDateTime = sh(
                        script: "date +%Y%m%d%H%M%S",
                        returnStdout: true
                    ).trim()

                    withCredentials([string(credentialsId: 'DOCKERHUB_TOKEN', variable: 'DOCKERHUB_TOKEN')]) {
                        docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
                            dir('bar-repo') {
                                sh """
                                echo "Building Docker image..."
                                docker build -t ${DOCKERHUB_REPO}:${currentDateTime} .

                                echo "Pushing Docker image to Docker Hub..."
                                docker push ${DOCKERHUB_REPO}:${currentDateTime}
                                """
                            }
                        }
                    }
                    // Save the image tag as an environment variable
                    env.IMAGE_TAG = currentDateTime
                }
            }
        }

        stage('Checkout CICD Repository and Update Deployment') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    script {
                        def cicdRepoUrl = CICD_REPO_URL.replace("https://", "https://${GITHUB_TOKEN}@")
        
                        if (fileExists('cicd-repo')) {
                            echo "CICD repository already exists. Pulling the latest changes..."
                            dir('cicd-repo') {
                                sh """
                                git reset --hard
                                git clean -fd
                                git pull origin main
                                """
                            }
                        } else {
                            echo "Cloning the CICD repository..."
                            sh "git clone ${cicdRepoUrl} cicd-repo"
                        }
        
                        dir('cicd-repo') {
                            sh """
                            echo "Updating ace-deployment.yaml with the new image tag..."
                            sed -i 's|image: preciousokpor/ibm-barfiles-image:.*|image: ${DOCKERHUB_REPO}:${env.IMAGE_TAG}|' ace-deployment.yaml
        
                            echo "Committing the updated ace-deployment.yaml..."
                            git config user.name "jenkins-bot"
                            git config user.email "jenkins@example.com"
                            git add ace-deployment.yaml
                            git commit -m "Update ace-deployment.yaml with image tag: ${env.IMAGE_TAG}"
                            git push origin main
                            """
                        }
                    }
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                script {
                    echo "Cleaning up local Docker image..."
                    sh """
                    docker rmi ${DOCKERHUB_REPO}:${env.IMAGE_TAG} || echo "Image already removed or not found."
                    """
                }
            }
        }
    }
}
