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
                    app = docker.build("dmnavarro/sample-jenkins-tmas")

                    // Test image
                    app.inside {
                        sh 'echo "Tests passed"'
                    }

                    // Push image with build number and latest tag
                    docker.withRegistry('https://registry.hub.docker.com', 'docker') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }

                    script {
                      def repo = 'dmnavarro/sample-jenkins-tmas'
                      def tag  = "${env.BUILD_NUMBER}"     // or whatever tag you actually pushed
                    
                      sh """
                        set -euo pipefail
                        docker pull ${repo}:${tag}
                        DIGEST=\$(docker inspect --format='{{index .RepoDigests 0}}' ${repo}:${tag} | cut -d'@' -f2)
                        echo "\$DIGEST" > image.digest
                      """
                      env.IMAGE_DIGEST = readFile('image.digest').trim()
                      echo "Captured digest: ${env.IMAGE_DIGEST}"
                    }
                }
            }
        }
        
        stage('Get Image Digest') {
            steps {
                script {
                    // Pull the latest image to get its digest
                    sh 'docker pull dmnavarro/sample-jenkins-tmas:latest'
                    def digest = sh(
                        script: "docker inspect --format='{{index .RepoDigests 0}}' dmnavarro/sample-jenkins-tmas:latest",
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
                    withCredentials([usernamePassword(credentialsId: 'github-pat', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                    }
                    
                    // Install TMAS
                    sh "mkdir -p $TMAS_HOME"
                    sh "curl -L https://cli.artifactscan.cloudone.trendmicro.com/tmas-cli/latest/tmas-cli_Linux_x86_64.tar.gz | tar xz -C $TMAS_HOME"
                    
                    // Execute the tmas scan command with the obtained digest
                    //sh 'cat ~/.docker/config.json'
                    sh "$TMAS_HOME/tmas scan -M -V -S registry:dmnavarro/sample-jenkins-tmas@${env.IMAGE_DIGEST} --region ap-southeast-1"
                    
                    // Logout from Docker Hub
                    sh 'docker logout'
                }
            }
        }

        stage('Deploy to EKS') {
          environment {
            AWS_REGION   = 'ap-southeast-1'
            EKS_CLUSTER  = 'drna_demo_c'
            K8S_NS       = 'default'
            DEPLOY_NAME  = 'web-app'
            IMAGE_REPO   = 'dmnavarro/sample-jenkins-tmas'
          }
          steps {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
              sh '''
                set -euo pipefail
            
                # Ensure kubectl exists (AL2023 x86_64, EKS 1.33)
                if ! command -v kubectl >/dev/null 2>&1; then
                  curl -sSLo kubectl curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.33.5/2025-09-19/bin/linux/amd64/kubectl
                  chmod +x kubectl
                  sudo mv kubectl /usr/local/bin/ || { echo "No sudo? Using local kubectl"; mv kubectl ./kubectl; export PATH="$PWD:$PATH"; }
                fi
            
                aws eks update-kubeconfig --name "${EKS_CLUSTER}" --region "${AWS_REGION}"
            
                # Apply base manifest (creates NS/Service/Deployment with dummy digest on first run)
                kubectl apply -f web-app.yaml
            
                # Patch the deployment to the EXACT image by digest you captured earlier
                kubectl -n "${K8S_NS}" set image deployment/${DEPLOY_NAME} ${DEPLOY_NAME}=${IMAGE_REPO}@${IMAGE_DIGEST}
            
                kubectl -n "${K8S_NS}" rollout status deployment/${DEPLOY_NAME} --timeout=5m
                kubectl -n "${K8S_NS}" get svc ${DEPLOY_NAME}
              '''
            }
        }
    }
    
    post {
        success {
            echo "Pipeline Complete"
        }
    }
}
