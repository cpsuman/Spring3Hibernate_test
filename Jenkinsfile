pipeline {
  agent any
  stages {
    stage('checkout') {
      steps {
        cleanWs(cleanWhenUnstable: true)
      }
    }
  }
}