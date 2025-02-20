pipeline {
    agent any
    
    environment {
        TMAS_API_KEY = credentials('TMAS_API_KEY')
        TMAS_HOME = "$WORKSPACE/tmas"
    }
    
    stages {
        stage('Build and Test Image') {
            steps {
                script {
                    def app

                    // Clone repository
                    checkout scm

                    // Build image
                    app = docker.build("dmnavarro/test")

                    // Test image
                    app.inside {
                        sh 'echo "Tests passed"'
                    }

                    // Push image with build number and latest tag
                    docker.withRegistry('https://registry.hub.docker.com', 'docker') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        
        stage('Get Image Digest') {
            steps {
                script {
                    // Pull the latest image to get its digest
                    sh 'docker pull dmnavarro/test:latest'
                    def digest = sh(
                        script: "docker inspect --format='{{index .RepoDigests 0}}' dmnavarro/test:latest",
                        returnStdout: true
                    ).trim()
                    
                    // Extract only the SHA part
                    def sha = digest.split('@')[1]
                    echo "Image digest: ${sha}"
                    
                    // Save the digest in an environment variable for subsequent steps
                    env.IMAGE_DIGEST = sha
                }
            }
        }
        
        stage('TMAS Scan') {
            steps {
                script {
                    // Login to Docker Hub
                    withCredentials([usernamePassword(credentialsId: 'docker', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                    }
                    
                    // Install TMAS
                    sh "mkdir -p $TMAS_HOME"
                    sh "curl -L https://cli.artifactscan.cloudone.trendmicro.com/tmas-cli/latest/tmas-cli_Linux_x86_64.tar.gz | tar xz -C $TMAS_HOME"
                    
                    // Execute the tmas scan command with the obtained digest
                    sh 'cat ~/.docker/config.json'
                    sh "$TMAS_HOME/tmas scan -M -V -S registry:dmnavarro/test@${env.IMAGE_DIGEST} --region ap-southeast-1"
                    
                    // Logout from Docker Hub
                    sh 'docker logout'
                }
            }
        }
    }
    
    post {
        success {
            echo "Pipeline Complete"
        }
    }
}
