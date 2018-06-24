pipeline {
  agent any
  stages {
    stage('build') {
      steps {
        sh './gradlew clean build copyAgent'
        stash(name: 'build-stash', includes: '**/build/**/*,Dockerfile,newrelic,runworker.sh')
        stash(name: 'aws-stash', includes: '**/aws/*')
      }
    }
    stage('publish') {
      parallel {
        stage('publish junit results') {
          steps {
            unstash 'build-stash'
            junit '**/build/test-results/**/*.xml'
          }
        }
        stage('publish artifacts') {
          steps {
            unstash 'build-stash'
            archiveArtifacts '**/build/libs/*.jar'
          }
        }
      }
    }
    stage('docker') {
      agent {
        node {
          label 'docker'
        }
        
      }
      steps {
        unstash 'build-stash'
        script {
          GIT_COMMIT_HASH = sh (script: "git rev-parse --short HEAD", returnStdout: true)
          
          sh "echo ${GIT_COMMIT_HASH}"
          sh "echo ${BUILD_TAG}"
          
          
          docker.withRegistry('https://1725*******4707.dkr.ecr.us-east-1.amazonaws.com','ecr:us-east-1:ecr-credentials') {
            def image = docker.build("cpo-ucl-load:cloudbees-test")
            image.push()
          }
        }
        
      }
    }
    stage('cloudformation-changeset') {
      steps {
        unstash 'aws-stash'
        script {
          withAWS(region: 'us-east-1', credentials: 'jenkins-username-password-credentials') {
            def outputs = cfnCreateChangeSet(stack:'cpo-ucl-load-alpha', changeSet:'test-change-set', file:'aws/cpo-ucl-batch.yaml', params:['ImageVersion=cloudbees-test'], keepParams:['DBUserName', 'EnvironmnetType', 'ImageName', 'RepositoryUri'], tags: ['test=test'],pollInterval:1000, roleArn:'arn:aws:iam::17****94707:role/jenkins-access')
          }
        }
        
      }
    }
  }
}
