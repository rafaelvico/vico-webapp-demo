# JENKINS FILE

def version, mvnCmd = "mvn -s configuration/cicd-settings-nexus3.xml"

pipeline {
  agent {
    label 'maven'
  }
  stages {
    stage('Build App') {
      steps {
        git branch: 'master', url: 'https://github.com/rafaelvico/vico-webapp-demo.git'
      }
    }
    stage('Build Image') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              openshift.selector("bc", "vwd").startBuild()
            }
          }
        }
      }
    }    
    stage('Deploy DEV') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              openshift.selector("dc", "vwd").rollout().latest();
            }
          }
        }
      }
    }
    stage('Promote to STAGE?') {
      steps {
        timeout(time:15, unit:'MINUTES') {
            input message: "Promote to STAGE?", ok: "Promote"
        }

        script {
          openshift.withCluster() {
            openshift.tag("${env.DEV_PROJECT}/vwd:latest", "${env.STAGE_PROJECT}/vwd:${version}")
          }
        }
      }
    }
    stage('Deploy STAGE') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.STAGE_PROJECT) {
              if (openshift.selector('dc', 'vwd').exists()) {
                openshift.selector('dc', 'vwd').delete()
                openshift.selector('svc', 'vwd').delete()
                openshift.selector('route', 'vwd').delete()
              }
              openshift.newApp("vwd:${version}").narrow("svc").expose()
              openshift.set("probe dc/vwd --readiness --get-url=http://:8080/ --initial-delay-seconds=30 --failure-threshold=10 --period-seconds=10")
              openshift.set("probe dc/vwd --liveness  --get-url=http://:8080/ --initial-delay-seconds=180 --failure-threshold=10 --period-seconds=10")
            }
          }
        }
      }
    }
  }
}
