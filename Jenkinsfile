pipeline {
  agent any
  environment {
    IMAGE_NAME = "samarthlavre"   // <-- change this
    CRED_ID    = "dockerhub-creds"                   // <-- ensure this credential exists in Jenkins
    TAG        = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          // try a few branch vars if available
          BR = env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'unknown'
          echo "Building branch: ${BR}"
        }
      }
    }

    stage('Build (Maven)') {
      steps {
        script {
          if (isUnix()) {
            sh 'mvn -B clean package -DskipTests=false'
          } else {
            bat 'mvn -B clean package -DskipTests=false'
          }
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
          junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('Docker: build image') {
      steps {
        script {
          if (isUnix()) {
            sh "docker build -t ${IMAGE_NAME}:${TAG} ."
          } else {
            bat "docker build -t ${IMAGE_NAME}:${TAG} ."
          }
        }
      }
    }

    stage('Docker: login & push') {
      steps {
        withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          script {
            if (isUnix()) {
              sh "echo $DH_PASS | docker login -u $DH_USER --password-stdin"
              sh "docker push ${IMAGE_NAME}:${TAG}"
            } else {
              // Windows: echo to file then docker login with --password-stdin
              bat """
                @echo %DH_PASS%>pw.txt
                type pw.txt | docker login -u %DH_USER% --password-stdin
                docker push ${IMAGE_NAME}:${TAG}
                del pw.txt
              """
            }
          }
        }
      }
    }

    stage('Deploy (run container on agent)') {
      steps {
        script {
          if (isUnix()) {
            sh """
              docker stop myapp || true
              docker rm myapp || true
              docker run -d --name myapp -p 8080:8080 ${IMAGE_NAME}:${TAG}
            """
          } else {
            bat """
              docker stop myapp || exit 0
              docker rm myapp || exit 0
              docker run -d --name myapp -p 8080:8080 ${IMAGE_NAME}:${TAG}
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "Pipeline succeeded: ${IMAGE_NAME}:${TAG}"
    }
    failure {
      echo "Pipeline failed"
    }
  }
}
