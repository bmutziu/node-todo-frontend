pipeline {
  environment {
    registry = "bmutziu/docker-test"
    registryCredential = 'dockerhub'
    dockerImage = ''
  }
  agent any
  tools { nodejs "node-kpi" }
  stages {
    stage('Setup') {
      steps {
        script {
          if (env.GIT_BRANCH == 'origin/develop' || env.GIT_BRANCH ==~ /(.+)feature-(.+)/) {
            target = 'dev'
          } else if (env.GIT_BRANCH ==~ /(.+)release-(.+)/) {
            target = 'pre'
          } else if (env.GIT_BRANCH == 'origin/master') {
            target = 'pro'
          } else {
            error "Unknown branch type: ${env.GIT_BRANCH}"
          }
          appversion  = sh(script: 'echo $(cat package.json | grep version | cut -d\' \' -f4 | sed \'s|[",]||g\')', returnStdout: true).trim()
          version     = appversion.take(10) + '-' + env.BUILD_NUMBER
          appname     = env.JOB_NAME
          prjname     = 'nodejs-todo-frontend'
          packname    = appname + '-v' + version + '.tar.gz'
          publish_url = env.NEXUS_URL + '/repository/raw-nodejs/' + prjname + '/' + packname
        }
      }
    }
    stage('Cloning Git') {
      steps {
        git 'https://github.com/bmutziu/node-todo-frontend.git'
      }
    }
    stage('Build') {
      steps {
        sh 'npm install'
      }
    }
    stage('APP Quality'){
      environment {
        scannerHome = tool 'sonarqube-nodejs'
        organization = "bmutziu-github"
      }
      steps {
	withSonarQubeEnv('sonarqube') {
            sh """${scannerHome}/bin/sonar-scanner \
                      -Dsonar.sources=src \
                      -Dsonar.organization=${organization} \
                      -Dsonar.projectKey=${env.JOB_NAME} \
                      -Dsonar.projectName=${env.JOB_NAME} \
                      -Dsonar.projectBaseDir=${env.WORKSPACE} \
                      -Dsonar.exclusions=src/test/**,node_modules/**,**/migrations/**,**/libs/transactional_db/** \
                      -Dsonar.sourceEncoding=UTF-8 \
                      -Dsonar.language=js \
                      -Dsonar.projectDescription='NodeJS ToDo FrontEnd' \
                      -Dsonar.projectVersion=${version}
               """
        }
      }
    }
    stage('Test') {
      steps {
	sh 'npm test'
	}
    }
    /*
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build registry + ":$BUILD_NUMBER"
        }
      }
    }
    stage('Deploy Image') {
      steps{
        script {
          docker.withRegistry( '', registryCredential ) {
            dockerImage.push()
          }
        }
      }
    }
    stage('Remove Unused docker image') {
      steps{
        sh "docker rmi $registry:$BUILD_NUMBER"
      }
    }
    */
  }
}
