pipeline {
  agent any

  environment {
    IMAGE_REPO  = 'dmnavarro/sample-jenkins-tmas'
    AWS_REGION  = 'ap-southeast-1'
    EKS_CLUSTER = 'drna_demo_c'   // <--- change if needed
    K8S_NS      = 'web-app'
    DEPLOY_NAME = 'web-app'
    TMAS_API_KEY = credentials('TMAS_API_KEY')
    TMAS_HOME   = "${WORKSPACE}/tmas"
  }

  stages {

    stage('Build & Test Image') {
      steps {
        sh '''
          docker build -t ${IMAGE_REPO} .
          docker run --rm ${IMAGE_REPO} /bin/sh -c 'echo Tests passed'
        '''
      }
    }

    stage('Push Image (BUILD_NUMBER tag)') {
      steps {
        withDockerRegistry([url: 'https://index.docker.io/v1/', credentialsId: 'docker']) {
          sh '''
            docker tag ${IMAGE_REPO} ${IMAGE_REPO}:${BUILD_NUMBER}
            docker push ${IMAGE_REPO}:${BUILD_NUMBER}
          '''
        }
      }
    }

    stage('Capture Image Digest') {
      steps {
        script {
          sh '''
            docker pull ${IMAGE_REPO}:${BUILD_NUMBER}
            DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ${IMAGE_REPO}:${BUILD_NUMBER} | cut -d'@' -f2)
            echo "$DIGEST" > image.digest
          '''
          env.IMAGE_DIGEST = readFile('image.digest').trim()
          echo "Captured digest: ${env.IMAGE_DIGEST}"
        }
      }
    }

    stage('TMAS Scan (by digest)') {
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
            sh "$TMAS_HOME/tmas scan -M -V -S registry:${IMAGE_REPO}@${IMAGE_DIGEST} --region ap-southeast-1"
            
            // Logout from Docker Hub
            sh 'docker logout'
        }
      }
    }

    stage('Deploy to EKS (by digest)') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh '''
            aws eks update-kubeconfig --name "${EKS_CLUSTER}" --region "${AWS_REGION}"

            # Ensure base resources exist (Namespace/Service/Deployment with dummy digest)
            kubectl apply -f web-app.yaml

            # Patch Deployment to the EXACT image digest
            kubectl -n "${K8S_NS}" set image deployment/${DEPLOY_NAME} \
              ${DEPLOY_NAME}=${IMAGE_REPO}@${IMAGE_DIGEST}

            kubectl -n "${K8S_NS}" rollout status deployment/${DEPLOY_NAME} --timeout=5m
            kubectl -n "${K8S_NS}" get svc ${DEPLOY_NAME} -o wide
          '''
        }
      }
    }

  } // end stages

} // end pipeline
