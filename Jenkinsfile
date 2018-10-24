pipeline {
  agent {
    label 'maven'
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
    stage('DT Deploy Event') {
      steps {
        createDynatraceDeploymentEvent(
          envId: 'Dynatrace Tenant',
          tagMatchRules: [
            [
              meTypes: [
                [meType: 'SERVICE']
              ],
              tags: [
                [context: 'CONTEXTLESS', key: 'environment', value: 'staging']
              ]
            ]
          ]) {

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
