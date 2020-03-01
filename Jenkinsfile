def appVersion="2.2.0.${BUILD_NUMBER}"


pipeline {
  agent {
    kubernetes {
      cloud 'openshift'
      label 'spring-petclinic-pipeline-agent'
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    worker: spring-petclinic-pipeline-agent
spec:
  containers:
  - name: jnlp
    image: registry.redhat.io/openshift4/ose-jenkins-agent-base:v4.2.15
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
  - name: java
    image: registry.redhat.io/codeready-workspaces/plugin-java11-rhel8:2.1
    command:
    - cat
    tty: true
  imagePullSecrets:
    - name: jenkins-pull-secret
"""
    }
  }

  options {
    timeout(time: 20, unit: 'MINUTES')
  }

  stages {
    stage('Build App') {
      steps {
        container("java"){
          sh "gradle bootJar -Pversion=${appVersion}"
        }
      }
    }
    stage('Test') {
      steps {
        container("java"){
          sh "gradle test jacocoTestReport"
        }
      }
      post{
        always{
          junit 'build/test-results/test/TEST-*.xml'
          jacoco execPattern: 'build/jacoco/test.exec'
        }
      }
    }
    stage('Code Analysis') {
      steps {
        script {
          container("java"){
            sh "gradle sonar"
          }
        }
      }
    }
    stage('Publish Jar') {
      steps {
        container("java"){
          sh "gradle publish -Pversion=${appVersion}"
        }
      }
    }
    stage('Build Image') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
            openshift.selector("bc", "petclinic").startBuild("--from-file=target/spring-petclinic-${appVersion}.jar", "--wait=true")
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
              openshift.selector("dc", "petclinic").rollout().latest()
            }
          }
        }
      }
    }
    stage(''){
      steps{
        script{
          def petclinicService = ''
          openshift.withCluster(){
            openshift.withProject(env.DEV_PROJECT) {
              petclinicService = openshift.selector("svc", "petclinic").object().get("spec").get("clusterIP")
            }
          }

          container("java"){
            sh "gradle webTest -Ptest.target.server.url=${petclinicService}"
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
            openshift.tag("${env.DEV_PROJECT}/petclinic:latest", "${env.STAGE_PROJECT}/petclinic:stage")
          }
        }
      }
    }
    stage('Deploy STAGE') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.STAGE_PROJECT) {
              openshift.selector("dc", "petclinic").rollout().latest()
            }
          }
        }
      }
    }
  }
}
