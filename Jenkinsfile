pipeline {

  agent {
    docker {
      image 'node:18-alpine'
      args '-v /var/run/docker.sock:/var/run/docker.sock'
    }
  }

  environment {
    DOCKER_IMAGE = "kamran623/node-k8s-app"
    DOCKER_TAG   = "${BUILD_NUMBER}"
    SONAR_HOST   = "http://localhost:9000"
  }

  stages {

    stage('1. Checkout Source Code') {
      steps {
        git branch: 'main',
            url: 'https://github.com/kamran90-web/my-node-app.git',
            credentialsId: 'github-creds'
      }
    }

    stage('2. Install Dependencies') {
      steps {
        sh 'npm install'
      }
    }

    stage('3. SonarQube Code Scan') {
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          sh """
            npx sonar-scanner \
              -Dsonar.projectKey=node-app \
              -Dsonar.sources=. \
              -Dsonar.host.url=${SONAR_HOST} \
              -Dsonar.login=$SONAR_TOKEN
          """
        }
      }
    }

    stage('4. Build Docker Image') {
      steps {
        sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
      }
    }

    stage('5. Push Docker Image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh """
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
            docker logout
          """
        }
      }
    }

    stage('6. Update Kubernetes Manifest Repo (ArgoCD)') {
      steps {
        git branch: 'main',
            url: 'https://github.com/kamran90-web/node-k8s-manifests.git',
            credentialsId: 'github-creds'

        sh """
          sed -i 's|image:.*|image: ${DOCKER_IMAGE}:${DOCKER_TAG}|' deployment.yaml
          git add deployment.yaml
          git commit -m "Update image to ${DOCKER_TAG}"
          git push origin main
        """
      }
    }
  }

  post {
    success {
      echo "✅ CI completed. ArgoCD will deploy automatically."
    }
    failure {
      echo "❌ Pipeline failed. Check logs."
    }
  }
}

