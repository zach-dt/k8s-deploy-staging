pipeline {
  agent {
    label 'maven'
  }
  environment {

  }
  stages {
    stage('Deploy to staging namespace') {
      steps {
        checkout scm
        container('kubectl') {
          sh "kubectl -n staging apply -f ."
        }
      }
    }
    stage('Deploy to production') {
      agent {
        label 'git'
      }
      steps {
        echo "deploy to prod"
      }
    }
  }
}
