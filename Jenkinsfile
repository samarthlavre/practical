pipeline {
  agent any
  environment {
    IMAGE_NAME = "samarthlavre" // change this
    CRED_ID    = "dockerhub-creds"                 // make sure this credential exists
    TAG        = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          echo "Workspace: ${pwd()}"
        }
      }
    }

    stage('Detect pom.xml') {
      steps {
        script {
          // uses Pipeline Utility Steps plugin (findFiles)
          def poms = findFiles(glob: '**/pom.xml')
          if (poms == null || poms.length == 0) {
            error "No pom.xml found in repository. Please ensure a Maven project exists."
          }
          def pomPath = poms[0].path
          env.PROJECT_DIR = pomPath.replaceAll(/\/pom.xml$/, '')
          if (env.PROJECT_DIR == '') {
            env.PROJECT_DIR = '.'
          }
          echo "Found pom.xml at: ${pomPath} -> project dir = ${env.PROJECT_DIR}"
        }
      }
    }

    stage('Build (Maven)') {
      steps {
        script {
          if (isUnix()) {
            sh "mvn -f ${env.PROJECT_DIR}/pom.xml -B clean package -DskipTests=false"
          } else {
            bat "\"%MAVEN_HOME%\\bin\\mvn\" -f \"${env.PROJECT_DIR}\\pom.xml\" -B clean package -DskipTests=false"
          }
        }
      }
      post {
        always {
          archiveArtifacts artifacts: "${env.PROJECT_DIR}/target/*.jar", allowEmptyArchive: true, fingerprint: true
          junit testResults: "${env.PROJECT_DIR}/target/surefire-reports/*.xml", allowEmptyResults: true
        }
      }
    }

    stage('Docker: build image') {
      steps {
        script {
          if (isUnix()) {
            sh "docker build -t ${IMAGE_NAME}:${TAG} ${env.PROJECT_DIR}"
          } else {
            bat "docker build -t ${IMAGE_NAME}:${TAG} ${env.PROJECT_DIR}"
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
  } // end stages

  post {
    success {
      echo "Pipeline succeeded: ${IMAGE_NAME}:${TAG}"
    }
    failure {
      echo "Pipeline failed"
    }
  }
} 
