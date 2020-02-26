def mvnCmd = "mvn -s configuration/cicd-settings-nexus3.xml"
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
  - name: maven
    image: registry.redhat.io/codeready-workspaces/stacks-java-rhel8:2.0
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
    stage('Set Version'){
      steps{
        container("maven"){
          sh "${mvnCmd} versions:set -DnewVersion=${appVersion}"
        }
      }
    }
    stage('Build App') {
      steps {
        container("maven"){
          sh "${mvnCmd} install -DskipTests=true"
        }
      }
    }
    stage('Test') {
      steps {
        container("maven"){
          sh "${mvnCmd} verify"
        }
      }
      post{
        always{
          junit '**/target/surefire-reports/TEST-*.xml'
          jacoco execPattern: 'target/jacoco.exec'
        }
      }
    }
    stage('Code Analysis') {
      steps {
        script {
          container("maven"){
            sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-sonarqube:9000 -DskipTests=true"
          }
        }
      }
    }
    stage('Archive App') {
      steps {
        container("maven"){
          sh "${mvnCmd} deploy -DskipTests=true"
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
              openshift.selector("dc", "petclinic").rollout().latest();
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
              openshift.selector("dc", "petclinic").rollout().latest();
            }
          }
        }
      }
    }
  }
}