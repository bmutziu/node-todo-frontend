pipeline {
  environment {
    registry = "bmutziu/docker-test"
    registryCredential = 'dockerhub'
    dockerImage = ''
    NEXUS_URL = "http://192.168.8.100:8081"
  }
  agent {
    label 'docker'
  }
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
        //organization = "bmutziu-github"
      }
      steps {
	withSonarQubeEnv('sonarqube') {
            echo "SONAR_HOST_URL=${SONAR_HOST_URL}"
            echo "SONAR_AUTH_TOKEN=${SONAR_AUTH_TOKEN}"
            sh """${scannerHome}/bin/sonar-scanner \
                      -Dsonar.sources=src \
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
    stage("Quality Gate") {
      steps {
        timeout(time: 16, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
      }
    }
    stage('Test') {
      steps {
	sh 'npm test'
	}
    }
    stage('APP Package'){
      steps {
        sh "tar --exclude='./.git' --exclude='./Jenkinsfile' --exclude='./Dockerfile' --exclude='./node_modules' -czv src/ -f " + packname
      }
    }
    stage('APP Publish') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-admin', passwordVariable: 'NEXUS_PASSWORD', usernameVariable: 'NEXUS_USERNAME')]) {
          sh 'curl -v -u $NEXUS_USERNAME:$NEXUS_PASSWORD --upload-file ' + packname + ' ' + publish_url
        }
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
    stage('DEV Deploy') {
      steps {
        sh 'echo Deploy to DEV . . .'
      }
    }

    stage('PRE Deploy') {
      when {
        expression { return target == 'pre' || target == 'pro' }
      }
      steps {
        sh 'echo Deploy to PRE . . .'
      }
    }

    stage('UAT Test') {
      when {
        expression { target == 'pro' }
      }
      steps {
        sh 'echo UAT Test . . .'
      }
    }

    stage('Approval') {
      when {
        expression { target == 'pro' }
      }
      steps {
        timeout(time:30, unit:'MINUTES') {
          input message: "Deploy to Production?", id: "approval"
        }
      }
    }

    stage('PRO Deploy') {
      when {
        expression { return target == 'pro' }
      }
      steps {
        sh 'echo Deploy to PRO . . .'
      }
    }
  }

  post {
    success {
      script {
        currentBuild.result = "SUCCESS"

        /* Custom data map for InfluxDB */
        def custom = [:]
        custom['branch']      = env.GIT_BRANCH
        custom['environment'] = target
        custom['part']        = 'jenkins'
        custom['version']     = version

        step([$class: 'InfluxDbPublisher', customData: custom, target: 'devops-kpi'], jenkinsEnvParameterField: 'build_agent_name=' + 'master' + '\n' + 'build_status_message=' + 'Just Words')
      }
    }

    failure {
      script {
        currentBuild.result = "FAILURE"

        /* Custom data map for InfluxDB */
        def custom = [:]
        custom['branch']      = env.GIT_BRANCH
        custom['environment'] = target
        custom['part']        = 'jenkins'
        custom['version']     = version

        step([$class: 'InfluxDbPublisher', customData: custom, target: 'devops-kpi'])
      }
    }

    unstable {
      script {
        currentBuild.result = "FAILURE"

        /* Custom data map for InfluxDB */
        def custom = [:]
        custom['branch']      = env.GIT_BRANCH
        custom['environment'] = target
        custom['part']        = 'jenkins'
        custom['version']     = version

        step([$class: 'InfluxDbPublisher', customData: custom, target: 'devops-kpi'])
      }
    }

     aborted {
      script {
        currentBuild.result = "FAILURE"

        /* Custom data map for InfluxDB */
        def custom = [:]
        custom['branch']      = env.GIT_BRANCH
        custom['environment'] = target
        custom['part']        = 'jenkins'
        custom['version']     = version

        step([$class: 'InfluxDbPublisher', customData: custom, target: 'devops-kpi'])
      }
    }
  }
  /*
  triggers {
    pollSCM('')
  }
  */
}
