pipeline {
  agent any
  environment {
    IMAGE_NAME = "samarthlavre" // CHANGE this
    CRED_ID    = "dockerhub-creds"                 // must exist in Jenkins
    TAG        = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script { echo "Workspace: ${pwd()}" }
      }
    }

    stage('Detect pom.xml') {
      steps {
        script {
          def poms = findFiles(glob: '**/pom.xml')
          if (!poms || poms.length == 0) {
            error "No pom.xml found in repository. Please ensure a Maven project exists."
          }
          def pomPath = poms[0].path
          def proj = pomPath.replaceAll(/^(?:\.?(?:\/|\\)?)?(.*?)(?:[\/\\]?pom\.xml)$/, '$1')
          env.PROJECT_DIR = (proj == null || proj.trim() == '' || proj.equalsIgnoreCase('pom.xml')) ? '.' : proj
          echo "Found pom.xml at: ${pomPath} -> project dir = ${env.PROJECT_DIR}"
          // normalized artifact/build dir
          env.ARTIFACT_DIR = (env.PROJECT_DIR == '.' ? 'target' : "${env.PROJECT_DIR.replaceAll('\\\\','/')}/target")
        }
      }
    }

    stage('Build (Maven)') {
      steps {
        script {
          if (isUnix()) {
            sh "mvn -f ${env.PROJECT_DIR}/pom.xml -B clean package -DskipTests=false"
          } else {
            bat "mvn -f \"${env.PROJECT_DIR}\\pom.xml\" -B clean package -DskipTests=false"
          }
        }
      }
      post {
        always {
          // use ARTIFACT_DIR to archive; allow empty so job doesn't fail on missing jars
          archiveArtifacts artifacts: "${env.ARTIFACT_DIR}/*.jar", allowEmptyArchive: true, fingerprint: true
          junit testResults: "${env.ARTIFACT_DIR}/surefire-reports/*.xml", allowEmptyResults: true
        }
      }
    }

    stage('Docker: build image (if available)') {
      steps {
        script {
          def dockerAvailable = false
          if (isUnix()) {
            dockerAvailable = sh(script: "docker info >/dev/null 2>&1 && echo OK || echo NO", returnStdout: true).trim() == 'OK'
          } else {
            def out = bat(script: 'docker info >nul 2>&1 && @echo OK || @echo NO', returnStdout: true).trim()
            dockerAvailable = out == 'OK'
          }

          if (!dockerAvailable) {
            echo "Docker daemon NOT available on this agent — skipping Docker build/push. Start Docker on the agent or run Docker on another host."
            currentBuild.description = "Docker skipped (not available)"
            // mark a flag so downstream stages can skip
            env.DOCKER_AVAILABLE = "false"
          } else {
            env.DOCKER_AVAILABLE = "true"
            if (isUnix()) {
              sh "docker build -t ${IMAGE_NAME}:${TAG} ${env.PROJECT_DIR}"
            } else {
              bat "docker build -t ${IMAGE_NAME}:${TAG} ${env.PROJECT_DIR}"
            }
          }
        }
      }
    }

    stage('Docker: login & push (if available)') {
      when { expression { env.DOCKER_AVAILABLE == "true" } }
      steps {
        withCredentials([usernamePassword(credentialsId: env.CRED_ID, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          script {
            if (isUnix()) {
              sh "echo $DH_PASS | docker login -u $DH_USER --password-stdin"
              sh "docker push ${IMAGE_NAME}:${TAG}"
            } else {
              bat "@echo %DH_PASS%>pw.txt & type pw.txt | docker login -u %DH_USER% --password-stdin & del pw.txt"
              bat "docker push ${IMAGE_NAME}:${TAG}"
            }
          }
        }
      }
    }

    stage('Deploy (run container on agent)') {
      when { expression { env.DOCKER_AVAILABLE == "true" } }
      steps {
        script {
          if (isUnix()) {
            sh "docker stop myapp || true; docker rm myapp || true; docker run -d --name myapp -p 8080:8080 ${IMAGE_NAME}:${TAG}"
          } else {
            bat "docker stop myapp || exit 0 & docker rm myapp || exit 0 & docker run -d --name myapp -p 8080:8080 ${IMAGE_NAME}:${TAG}"
          }
        }
      }
    }
  }

  post {
    success { echo "Pipeline succeeded: ${IMAGE_NAME}:${TAG}" }
    failure { echo "Pipeline failed" }
  }
}
