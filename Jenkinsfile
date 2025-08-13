pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '15'))
  }

  parameters {
    choice(name: 'ENV', choices: ['staging', 'prod'], description: 'Ambiente de despliegue')
  }

  environment {
    // Ajusta el nombre del JDK configurado en Jenkins (Global Tool Configuration)
    JAVA_HOME = tool name: 'JDK21', type: 'hudson.model.JDK'
    PATH = "${JAVA_HOME}/bin:${env.PATH}"
    // Si usas Maven desde Tools:
    MAVEN_HOME = tool name: 'Maven3', type: 'hudson.tasks.Maven$MavenInstallation'
    M2_HOME = "${MAVEN_HOME}"
    DEPLOY_HOST = credentials('deploy-host') // opcional: host/usuario por credencial tipo "Username/Password"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build & Test') {
      steps {
        sh """
          ${M2_HOME}/bin/mvn -B -Dmaven.test.failure.ignore=false clean verify
        """
      }
      post {
        always {
          junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('Package') {
      steps {
        sh """
          ${M2_HOME}/bin/mvn -B package -DskipTests
        """
      }
      post {
        success {
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
      }
    }

    stage('Deploy to Staging') {
      when { expression { params.ENV == 'staging' } }
      steps {
        // Requiere plugin "SSH Agent" y una credencial de tipo "SSH Username with private key" con ID: 'ssh-deploy-key'
        sshagent(credentials: ['ssh-deploy-key']) {
          sh '''
            set -e
            APP_JAR=$(ls target/*.jar | head -n1)
            scp -o StrictHostKeyChecking=no "$APP_JAR" ${DEPLOY_HOST_USR}@${DEPLOY_HOST_PSW}:/opt/apps/myapp/app.jar
            ssh -o StrictHostKeyChecking=no ${DEPLOY_HOST_USR}@${DEPLOY_HOST_PSW} '
              sudo systemctl stop myapp || true
              sudo mv /opt/apps/myapp/app.jar /opt/apps/myapp/current.jar
              sudo systemctl start myapp
              systemctl status myapp --no-pager -l
            '
          """
        }
      }
    }
  }

  post {
    success { echo "✅ Build OK: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
    failure { echo "❌ Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
  }
}
