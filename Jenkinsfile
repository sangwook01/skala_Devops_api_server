pipeline {
  agent any

  environment {
    APP_DIR                 = '01.spring-backend-v1.0'
    GIT_URL                 = 'https://github.com/sangwook01/skala_Devops_api_server.git'
    GIT_BRANCH              = 'main'
    GIT_ID                  = 'sk006-github-id'
    IMAGE_REGISTRY_URL      = 'skala-registry.skala-ai.com'
    IMAGE_REGISTRY_PROJECT  = 'class-1'
    IMAGE_NAME              = 'sk006-webserver'
    IMAGE_TAG_BASE          = '1.0-buildah'
    REGISTRY_CREDENTIAL_ID  = 'skala-image-registry-id'
    K8S_NAMESPACE           = 'class-1'
    DEPLOYMENT_NAME         = 'sk006-webserver'
  }

  options {
    disableConcurrentBuilds()
    timestamps()
    timeout(time: 20, unit: 'MINUTES')
  }

  stages {
    stage('Clone Repository') {
      steps {
        git branch: "${GIT_BRANCH}", url: "${GIT_URL}", credentialsId: "${GIT_ID}"
      }
    }

    stage('Build with Maven') {
      steps {
        dir("${APP_DIR}") {
          sh 'mvn -s repo-setting/settings.xml clean package -DskipTests'
        }
      }
    }

    stage('Image Build and Push - Buildah') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${REGISTRY_CREDENTIAL_ID}",
          usernameVariable: 'REGISTRY_USER',
          passwordVariable: 'REGISTRY_PASSWORD'
        )]) {
          sh '''
            set -eu
            FINAL_IMAGE_TAG="${IMAGE_TAG_BASE}-${BUILD_NUMBER}"
            IMAGE_REF="${IMAGE_REGISTRY_URL}/${IMAGE_REGISTRY_PROJECT}/${IMAGE_NAME}:${FINAL_IMAGE_TAG}"
            sudo buildah login -u "$REGISTRY_USER" -p "$REGISTRY_PASSWORD" "$IMAGE_REGISTRY_URL"
            sudo buildah bud --storage-driver=vfs --isolation=chroot --platform linux/amd64 -t "$IMAGE_REF" "$APP_DIR"
            sudo buildah push --storage-driver=vfs "$IMAGE_REF"
            sudo buildah rmi "$IMAGE_REF" || true
          '''
        }
      }
    }

    stage('Update Kustomize') {
      steps {
        sh '''
          set -eu
          FINAL_IMAGE_TAG="${IMAGE_TAG_BASE}-${BUILD_NUMBER}"
          IMAGE_BASE="${IMAGE_REGISTRY_URL}/${IMAGE_REGISTRY_PROJECT}/${IMAGE_NAME}"
          cd "${APP_DIR}/kustomize/overlays/dev"
          kustomize edit set image "${IMAGE_BASE}=${IMAGE_BASE}:${FINAL_IMAGE_TAG}"
        '''
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh '''
          set -eu
          kustomize build "${APP_DIR}/kustomize/overlays/dev" | kubectl apply -n "$K8S_NAMESPACE" -f -
          kubectl rollout status "deployment/${DEPLOYMENT_NAME}" -n "$K8S_NAMESPACE" --timeout=180s
          kubectl get deployment,pod,service -n "$K8S_NAMESPACE" -l "app=${DEPLOYMENT_NAME}"
        '''
      }
    }
  }

  post {
    always {
      deleteDir()
    }
  }
}
