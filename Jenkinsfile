import groovy.transform.Field

jsl = library(
  identifier: "jsl-peanut-butter@1.0.3",
  retriever: modernSCM(
    [
      $class: 'GitSCMSource',
      remote: 'https://github.com/jflowers/jsl-peanut-butter.git',
      credentialsId: 'github'
    ]
  )
)

@Field
def appName = 'pipeline'

@Field
def versionPrefix = '2.2.0'

def getgradleCmd(){
  return "./gradlew -ParchiveVersion=${version.buildVersion}"
}

def nodeLabel = 'spring-petclinic-pipeline-agent'

version = jsl.com.peanutbutter.jenkins.Version.new(this, versionPrefix)

pipeline {
  agent {
    kubernetes {
      cloud 'openshift'
      label nodeLabel
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    worker: ${nodeLabel}
spec:
  containers:
  - name: jnlp
    image: registry.redhat.io/openshift4/ose-jenkins-agent-base:v4.2.15
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
  - name: java
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
    stage('Build App') {
      steps {
        container("java"){
          sh "${gradleCmd} bootJar"
        }
      }
    }
    stage('Test') {
      steps {
        container("java"){
          sh "${gradleCmd} test jacocoTestReport"
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
        container("java"){
          sh "${gradleCmd} sonar"
        }
      }
    }
    stage('Publish Jar') {
      steps {
        container("java"){
          sh "${gradleCmd} -Dorg.gradle.internal.publish.checksums.insecure=true"
        }
      }
    }
    stage('Build Image') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
            openshift.selector("bc", "petclinic").startBuild("--from-file=build/libs/spring-petclinic-${version.buildVersion}.jar", "--wait=true")
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
    stage('Web Testing'){
      steps{
        container("java"){
          sh "${gradleCmd} webTest"
        }
      }
      post{
        always{
          junit 'build/test-results/webTest/TEST-*.xml'
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

  post{
    always{
      script{
        def crwHost = ''
        openshift.withCluster() {
          crwHost = openshift.selector("route", "codeready").object().get("spec").get("host")
        }

        def summary = createSummary(icon: 'notepad.gif')
        summary.appendText("<h2><a href=\"https://${crwHost}/f?url=http://gogs.appdev-opentlc.svc.cluster.local:3000/gogs/spring-petclinic/raw/master/devfile.yaml\" target=\"_blank\" rel=\"noopener noreferrer\">Create a CodeReady Worskpace</a></h2>", false)
      }
    }
  }
}