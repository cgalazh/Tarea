pipeline {
  agent any

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '15'))
  }

  parameters {
    choice(name: 'ENV', choices: ['staging', 'prod'], description: 'Ambiente de despliegue')
  }

  environment {
    JAVA_HOME = tool name: 'JDK21', type: 'hudson.model.JDK'
    PATH = "${JAVA_HOME}/bin:${env.PATH}"
    MAVEN_HOME = tool name: 'Maven3', type: 'hudson.tasks.Maven$MavenInstallation'
    M2_HOME = "${MAVEN_HOME}"
    DEPLOY_HOST = credentials('deploy-host') // Username/Password → crea env.DEPLOY_HOST_USR/PSW
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Test') {
      steps {
        sh """
          ${M2_HOME}/bin/mvn -B -Dmaven.test.failure.ignore=false clean verify
        """
      }
      post { always { junit 'target/surefire-reports/*.xml' } }
    }

    stage('Package') {
      steps {
        sh """
          ${M2_HOME}/bin/mvn -B package -DskipTests
        """
      }
      post { success { archiveArtifacts artifacts: 'target/*.jar', fingerprint: true } }
    }

    stage('Deploy to Staging') {
      when { expression { params.ENV == 'staging' } }
      steps {
        sshagent(credentials: ['ssh-deploy-key']) {
          script {
            def host = "${env.DEPLOY_HOST_USR}@${env.DEPLOY_HOST_PSW}"
            sh '''
              set -e
              APP_JAR=$(ls target/*.jar | head -n1)
              echo "Subiendo: $APP_JAR"
            '''
            sh "scp -o StrictHostKeyChecking=no target/*.jar ${host}:/opt/apps/myapp/app.jar"
            sh "ssh -o StrictHostKeyChecking=no ${host} 'sudo systemctl stop myapp || true; sudo mv /opt/apps/myapp/app.jar /opt/apps/myapp/current.jar || true; sudo systemctl start myapp; systemctl status myapp --no-pager -l'"
          }
        }
      }
    }
  }

  post {
    success { echo "✅ Build OK: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
    failure { echo "❌ Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
  }
}
