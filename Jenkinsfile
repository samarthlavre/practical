pipeline {
  agent any
  environment {
    IMAGE_NAME = "samarth" // <-- change this
    CRED_ID    = "dockerhub-creds"                 // <-- ensure this exists in Jenkins
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
          // findFiles requires Pipeline Utility Steps plugin
          def poms = findFiles(glob: '**/pom.xml')
          if (poms == null || poms.length == 0) {
            error "No pom.xml found in repository. Please ensure a Maven project exists."
          }
          // choose the first pom found
          def pomPath = poms[0].path
          env.PROJECT_DIR = pomPath.replaceAll(/\/pom.xml$/, '')
          if (env.PROJECT_DIR == '') {
            env.PROJECT_DIR = '.' // root
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
            // Windows (bat) - use double quotes to handle spaces
            bat "\"%MAVEN_HOME%\\bin\\mvn\" -f \"${env.PROJECT_DIR}\\pom.xml\" -B clean package -DskipTests=false"
          }
        }
      }
      post {
        always {
          // archive any jars if produced; allow empty to avoid failing
          archiveArtifacts artifacts: "${env.PROJECT_DIR}/target/*.jar", allowEmptyArchive: true, fingerprint: true
          // record junit if present; allow empty so it doesn't error
          junit testResults: "${env.PROJECT_DIR}/target/surefire-reports/*.xml", allowEmptyResults: true
        }
      }
    }
