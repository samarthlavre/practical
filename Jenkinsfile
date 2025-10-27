@"
pipeline {
  agent any
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Echo') {
      steps {
        echo "Jenkinsfile found and pipeline running on branch ${env.BRANCH_NAME}"
      }
    }
  }
}
"@ | Set-Content -Path .\Jenkinsfile -Encoding UTF8
