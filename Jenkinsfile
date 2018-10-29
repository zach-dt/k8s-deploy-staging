pipeline {
  parameters {
    string(name: 'APP_NAME', defaultValue: '', description: 'The name of the service to deploy.', trim: true)
    string(name: 'TAG_STAGING', defaultValue: '', description: 'The image of the service to deploy.', trim: true)
    string(name: 'VERSION', defaultValue: '', description: 'The version of the service to deploy.', trim: true)
  }
  agent {
    label 'maven'
  }
  stages {
    stage('Update Deployment and Service specification') {
      agent {
        label 'git'
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'git-credentials-acm', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
          sh "git config --global user.email ${env.GITHUB_USER_EMAIL}"
          //sh "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${env.GITHUB_ORGANIZATION}/k8s-deploy-staging"
          sh "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/dynatrace-sockshop/k8s-deploy-staging"
          sh "cd k8s-deploy-staging/ && sed -i 's#image: .*#image: ${env.TAG_STAGING}#'  ${env.APP_NAME}.yml"
          sh "cd k8s-deploy-staging/ && git add ${env.APP_NAME}.yml && git commit -m 'Update ${env.APP_NAME} version ${env.VERSION}'"
          //sh "cd k8s-deploy-staging/ && git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${env.GITHUB_ORGANIZATION}/k8s-deploy-staging"
          sh "cd k8s-deploy-staging/ && git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/dynatrace-sockshop/k8s-deploy-staging"
        }
      }
    }
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
                [context: 'CONTEXTLESS', key: 'app', value: "${env.APP_NAME}"],
                [context: 'CONTEXTLESS', key: 'environment', value: 'staging']
              ]
            ]
          ]) {
        }
      }
    }
    stage('Performance Check Staging') {
      agent {
        label 'git'
      }
      steps {
        recordDynatraceSession(
          envId: 'Dynatrace Tenant',
          testCase: 'loadtest',
          tagMatchRules: [
            [
              meTypes: [
                [meType: 'SERVICE']
              ],
              tags: [
                [context: 'CONTEXTLESS', key: 'app', value: "${env.APP_NAME}"],
                [context: 'CONTEXTLESS', key: 'environment', value: 'staging']
              ]
            ]
          ]) {
          build job: "jmeter-tests",
            parameters: [
              string(name: 'SCRIPT_NAME', value: "${env.APP_NAME}_load.jmx"),
              string(name: 'SERVER_URL', value: "${env.APP_NAME}.staging"),
              string(name: 'SERVER_PORT', value: '80'),
              string(name: 'CHECK_PATH', value: '/health'),
              string(name: 'VUCount', value: '1'), // 10
              string(name: 'LoopCount', value: '2'), // 250
              string(name: 'DT_LTN', value: "PerfCheck_${BUILD_NUMBER}"),
              string(name: 'FUNC_VALIDATION', value: 'no'),
              string(name: 'AVG_RT_VALIDATION', value: '250')
            ]
        }
        sh "git clone https://github.com/dynatrace-sockshop/${env.APP_NAME}"
        // Now we use the Performance Signature Plugin to pull in Dynatrace Metrics based on the spec file
        perfSigDynatraceReports envId: 'Dynatrace Tenant', nonFunctionalFailure: 1, specFile: "${env.APP_NAME}/monspec/${env.APP_NAME}_perfsig.json"
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
