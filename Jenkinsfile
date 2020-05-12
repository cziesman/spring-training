#!/usr/bin/groovy
pipeline {

  agent {
      label 'maven'
  }

  stages {

    stage('Build App') {
      steps {
        git url: "https://github.com/cziesman/spring-training.git"
        sh "mvn install"
        stash name:"jar", includes:"target/spring-training-1.0.0-SNAPSHOT.jar"
      }
    }

    stage('Build Image') {
      steps {
        unstash name:"jar"
        sh "oc start-build spring-training --from-file=target/spring-training-1.0.0-SNAPSHOT.jar --follow"
      }
    }

    stage('Promote to DEV') {
      steps {
        script {
          openshift.withCluster() {
            openshift.tag("spring-training:latest", "spring-training:dev")
          }
        }
      }
    }

    stage('Create DEV') {
//      when {
//        expression {
//          openshift.withCluster() {
//            return !openshift.selector('dc', 'spring-training').exists()
//          }
//        }
//      }

      steps {
        script {
//            openshiftDeploy depCfg: 'spring-training'
//            openshiftVerifyDeployment depCfg: 'spring-training', replicaCount: 1, verifyReplicaCount: true
          openshift.withCluster() {
            openshift.newApp("spring-training:latest", "--name=spring-training").narrow('svc').expose()
          }
        }
      }
    }

    stage('Promote STAGE') {
      steps {
        script {
          openshift.withCluster() {
            openshift.tag("spring-training:dev", "spring-training:stage")
          }
        }
      }
    }

    stage('Create STAGE') {
      when {
        expression {
          openshift.withCluster() {
            return !openshift.selector('dc', 'spring-training-stage').exists()
          }
        }
      }

      steps {
        script {
          openshift.withCluster() {
            openshift.newApp("spring-training:stage", "--name=spring-training-stage").narrow('svc').expose()
          }
        }
      }
    }
  }
}
