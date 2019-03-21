@Library('dynatrace@master') _

def tagMatchRules = [
  [
    meTypes: [
      [meType: 'SERVICE']
    ],
    tags : [
      [context: 'CONTEXTLESS', key: 'app', value: ''],
      [context: 'CONTEXTLESS', key: 'environment', value: 'staging']
    ]
  ]
]

pipeline {
  parameters {
    string(name: 'APP_NAME', defaultValue: '', description: 'The name of the service to deploy.', trim: true)
    string(name: 'TAG_STAGING', defaultValue: '', description: 'The image of the service to deploy.', trim: true)
    string(name: 'VERSION', defaultValue: '', description: 'The version of the service to deploy.', trim: true)
  }
  agent {
    label 'kubegit'
  }
  stages {
    stage('Update Deployment and Service specification') {
      steps {
        container('git') {
          withCredentials([usernamePassword(credentialsId: 'git-credentials-acm', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
            sh "git config --global user.email ${env.GITHUB_USER_EMAIL}"
            sh "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${env.GITHUB_ORGANIZATION}/k8s-deploy-staging"
            sh "cd k8s-deploy-staging/ && sed -i 's#image: .*#image: ${env.TAG_STAGING}#' ${env.APP_NAME}.yml"
            sh "cd k8s-deploy-staging/ && git add ${env.APP_NAME}.yml && git commit -m 'Update ${env.APP_NAME} version ${env.VERSION}'"
            sh "cd k8s-deploy-staging/ && git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${env.GITHUB_ORGANIZATION}/k8s-deploy-staging"
          }
        }
      }
    }
    stage('Deploy to staging namespace') {
      steps {
        checkout scm
        container('kubectl') {
          sh "kubectl -n staging apply -f ${env.APP_NAME}.yml"
        }
      }
    }
    // DO NOT uncomment until 06_04 Lab
    
    stage('DT Deploy Event') {
      steps {
        container("curl") {
          script {
            tagMatchRules[0].tags[0].value = "${env.APP_NAME}"
            def status = pushDynatraceDeploymentEvent (
              tagRule : tagMatchRules,
              customProperties : [
                [key: 'Jenkins Build Number', value: "${env.BUILD_ID}"],
                [key: 'Git commit', value: "${env.GIT_COMMIT}"]
              ]
            )
          }
        }
      }
    }
    
    
    // DO NOT uncomment until 10_01 Lab
    stage('Staging Warm Up') {
          steps {
            echo "Waiting for the service to start..."
            container('kubectl') {
              script {
                def result=1
                def i=0
                while (result!=0 && i < 30) {
                  //command will return status code 1 if deployment is not ready
                  result=sh(script: "kubectl -n staging rollout status deployment ${env.APP_NAME} --timeout=10s", returnStatus: true)
                  i++;
                }
                // If we reach the end of the loop (5mins) and the deployment is still not ready, Fail the build
                if(result!=0) {
                  currentBuild.result = 'FAILED'
                  error "Error while waiting for ${env.APP_NAME} deployment to become available."
                }
              }
            }
            echo "Running one iteration with one VU to warm up service"  
            container('jmeter') {
              script {
                def status = executeJMeter ( 
                  scriptName: "jmeter/front-end_e2e_load.jmx",
                  resultsDir: "e2eCheck_${env.APP_NAME}_warmup",
                  serverUrl: "front-end.staging", 
                  serverPort: 8080,
                  checkPath: '/health',
                  vuCount: 1,
                  loopCount: 1,
                  LTN: "e2eCheck_${BUILD_NUMBER}_warmup",
                  funcValidation: false,
                  avgRtValidation: 4000
                )
                if (status != 0) {
                  currentBuild.result = 'FAILED'
                  error "Warm up round in staging failed."
                }
              }
            }
          }
        }


    stage('Run production ready e2e check in staging') {
      steps {
        echo "Waiting for the service to start..."
        // sleep 150

        recordDynatraceSession (
          envId: 'Dynatrace Tenant',
          testCase: 'loadtest',
          tagMatchRules: tagMatchRules
        ) 
        {
          container('jmeter') {
            script {
              def status = executeJMeter ( 
                scriptName: "jmeter/front-end_e2e_load.jmx",
                resultsDir: "e2eCheck_${env.APP_NAME}",
                serverUrl: "front-end.staging", 
                serverPort: 8080,
                checkPath: '/health',
                vuCount: 10,
                loopCount: 5,
                LTN: "e2eCheck_${BUILD_NUMBER}",
                funcValidation: false,
                avgRtValidation: 4000
              )
              if (status != 0) {
                currentBuild.result = 'FAILED'
                error "Production ready e2e check in staging failed."
              }
            }
          }
        }

        perfSigDynatraceReports(
          envId: 'Dynatrace Tenant', 
          nonFunctionalFailure: 1, 
          specFile: "monspec/e2e_perfsig.json"
        )
      }
    }
    
  }
}
