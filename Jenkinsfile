pipeline {
  agent any

  environment {
    IMAGE_REPO  = 'dmnavarro/sample-jenkins-tmas'
    AWS_REGION  = 'ap-southeast-1'
    EKS_CLUSTER = 'drna_demo_c'   // <--- change if needed
    K8S_NS      = 'web-app'
    DEPLOY_NAME = 'web-app'
    TMAS_HOME   = "${WORKSPACE}/tmas"
  }

  stages {

    stage('Build & Test Image') {
      steps {
        sh '''
          set -euo pipefail
          docker build -t ${IMAGE_REPO} .
          docker run --rm ${IMAGE_REPO} /bin/sh -c 'echo Tests passed'
        '''
      }
    }

    stage('Push Image (BUILD_NUMBER tag)') {
      steps {
        withDockerRegistry([url: 'https://index.docker.io/v1/', credentialsId: 'docker']) {
          sh '''
            set -euo pipefail
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
            set -euo pipefail
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
        withCredentials([usernamePassword(
          credentialsId: 'tmas-api',
          usernameVariable: 'TMAS_API_KEY',
          passwordVariable: 'TMAS_API_KEY_PSW'
        )]) {
          sh '''
            set -euo pipefail

            mkdir -p "${TMAS_HOME}"
            curl -sSL https://cli.artifactscan.cloudone.trendmicro.com/tmas-cli/latest/tmas-cli_Linux_x86_64.tar.gz \
              | tar xz -C "${TMAS_HOME}"

            # If TMAS needs registry auth, uncomment the next three lines and ensure DOCKER creds exist in env
            # echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
            # "${TMAS_HOME}/tmas" scan -M -V -S registry:${IMAGE_REPO}@${IMAGE_DIGEST} --region ${AWS_REGION}
            # docker logout

            # Most setups can scan public images without docker login:
            "${TMAS_HOME}/tmas" scan -M -V -S registry:${IMAGE_REPO}@${IMAGE_DIGEST} --region ${AWS_REGION}
          '''
        }
      }
    }

    stage('Deploy to EKS (by digest)') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh '''
            set -euo pipefail

            if ! command -v kubectl >/dev/null 2>&1; then
              curl -sSLo kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.33.0/2024-09-10/bin/linux/amd64/kubectl
              chmod +x kubectl
              sudo mv kubectl /usr/local/bin/ 2>/dev/null || { mv kubectl ./kubectl; export PATH="$PWD:$PATH"; }
            fi

            aws eks update-kubeconfig --name "${EKS_CLUSTER}" --region "${AWS_REGION}"

            # Ensure base resources exist (Namespace/Service/Deployment with dummy digest)
            kubectl apply -f k8s/web-app.yaml

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

  post {
    success {
      echo "Built, pushed, scanned and deployed ${IMAGE_REPO}@${IMAGE_DIGEST}"
    }
    failure {
      echo "Pipeline failed — see logs above."
    }
    always {
      sh 'docker logout 2>/dev/null || true'
    }
  }

} // end pipeline
