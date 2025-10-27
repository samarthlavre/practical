pipeline {
  agent any
  environment {
    IMAGE_NAME = "samarthlavre" // change this
    CRED_ID    = "dockerhub-creds"                 // ensure this credential exists
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
          archiveArtifacts artifacts: "${env.ARTIFACT_DIR}/*.jar", allowEmptyArchive: true, fingerprint: true
          junit testResults: "${env.ARTIFACT_DIR}/surefire-reports/*.xml", allowEmptyResults: true
        }
      }
    }

    stage('Docker: detect') {
      steps {
        script {
          if (isUnix()) {
            def rc = sh(returnStatus: true, script: 'docker info >/dev/null 2>&1')
            env.DOCKER_AVAILABLE = (rc == 0) ? "true" : "false"
          } else {
            def rc = bat(returnStatus: true, script: '@echo off & docker info >nul 2>&1')
            env.DOCKER_AVAILABLE = (rc == 0) ? "true" : "false"
          }
          echo "Docker available: ${env.DOCKER_AVAILABLE}"
        }
      }
    }

    stage('Docker: build image') {
      when { expression { env.DOCKER_AVAILABLE == "true" } }
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

    stage('Deploy (run container)') {
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
