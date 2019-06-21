#!/usr/bin/env groovy

pipeline {
  agent {
      kubernetes {
          label "esaco-${env.JOB_BASE_NAME}-${env.BUILD_NUMBER}"
          cloud 'Kube mwdevel'
          defaultContainer 'jnlp'
          inheritFrom 'ci-template'
      }
  }
  
  options {
    timeout(time: 1, unit: 'HOURS')
    buildDiscarder(logRotator(numToKeepStr: '5'))
  }
  
  stages {

    stage('build') {
      steps {
        container('runner'){
          sh 'mvn -B clean compile'
        }
      }
    }

    stage('test') {
      steps {
        container('runner'){
          sh 'mvn -B clean test'
        }
      }

      post {
        always {
          container('runner'){
            junit '**/target/surefire-reports/TEST-*.xml'
            jacoco()
          }
        }
      }
    }
    
    stage('package') {
      steps {
        container('runner'){
          sh 'mvn -DskipTests clean package'
          stash includes: 'esaco-app/target/esaco-app-*.jar', name: 'esaco-artifacts'
          sh 'sh utils/print-pom-version.sh > esaco-version'
          stash includes: 'esaco-version', name: 'esaco-version'
        }
      }
    }

    stage('PR analysis'){
      when{
        not {
          environment name: 'CHANGE_URL', value: ''
        }
      }
      steps {
        container('runner'){
          script{
            def tokens = "${env.CHANGE_URL}".tokenize('/')
            def organization = tokens[tokens.size()-4]
            def repo = tokens[tokens.size()-3]

            withCredentials([string(credentialsId: '630f8e6c-0d31-4f96-8d82-a1ef536ef059', variable: 'GITHUB_ACCESS_TOKEN')]) {
              withSonarQubeEnv{
                sh """
                  mvn -B -U clean package -DskipTests sonar:sonar \\
                    -Dsonar.analysis.mode=preview \\
                    -Dsonar.github.pullRequest=${env.CHANGE_ID} \\
                    -Dsonar.github.repository=${organization}/${repo} \\
                    -Dsonar.github.oauth=${GITHUB_ACCESS_TOKEN} \\
                    -Dsonar.host.url=${SONAR_HOST_URL} \\
                    -Dsonar.login=${SONAR_AUTH_TOKEN}
                """
              }
            }
          }
        }
      }
    }
    
    stage('analysis'){
      when{
        anyOf { branch 'master'; branch 'develop' }
        environment name: 'CHANGE_URL', value: ''
      }
      steps {
        container('runner'){
          script{
            def opts = '-Dmaven.test.failure.ignore -DfailIfNoTests=false -DskipTests'
            def checkstyle_opts = 'checkstyle:check -Dcheckstyle.config.location=google_checks.xml'

            withSonarQubeEnv{
              sh "mvn clean package -U ${opts} ${checkstyle_opts} ${SONAR_MAVEN_GOAL} -Dsonar.host.url=${SONAR_HOST_URL} -Dsonar.login=${SONAR_AUTH_TOKEN}"
            }
          }
        }
      }
    }
    
    stage('deploy') {
      steps {
        container('runner'){
          sh "mvn clean -DskipTests -U -B deploy"
        }
      }
    }
    
    stage('docker-images') {
      steps {
        container('runner') {
          unstash 'esaco-artifacts'
          unstash 'esaco-version'
          sh'''
          POM_VERSION=$(cat esaco-version) /bin/bash esaco-app/docker/build-image.sh
          POM_VERSION=$(cat esaco-version) /bin/bash esaco-app/docker/push-image.sh
          '''
          script {
            if (env.BRANCH_NAME == 'master') {
              sh '''
              unset DOCKER_REGISTRY_HOST
              POM_VERSION=$(cat esaco-version) /bin/bash esaco-app/docker/push-image.sh
              '''
            }
          }
        }
      }
    }

    stage('result'){
      steps {
        script { currentBuild.result = 'SUCCESS' }
      }
    }
  }
  
  post {
    failure {
      slackSend color: 'danger', message: "${env.JOB_NAME} - #${env.BUILD_NUMBER} Failure (<${env.BUILD_URL}|Open>)"
    }
    
    unstable {
      slackSend color: 'warning', message: "${env.JOB_NAME} - #${env.BUILD_NUMBER} Unstable (<${env.BUILD_URL}|Open>)"
    }
    
    changed {
      script{
        if('SUCCESS'.equals(currentBuild.result)) {
          slackSend color: 'good', message: "${env.JOB_NAME} - #${env.BUILD_NUMBER} Back to normal (<${env.BUILD_URL}|Open>)"
        }
      }
    }
  }
}
