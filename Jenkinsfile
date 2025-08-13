pipeline {
  agent any

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '15'))
  }

  parameters {
    choice(name: 'ENV', choices: ['staging', 'prod'], description: 'Ambiente de despliegue')
    string(name: 'DEPLOY_TARGET', defaultValue: 'usuario@servidor', description: 'Destino SSH, ej: user@host')
  }

  environment {
    // Ajusta los nombres de Tools a los que tengas en Manage Jenkins > Global Tool Configuration
    JAVA_HOME = tool name: 'JDK21', type: 'hudson.model.JDK'
    PATH = "${JAVA_HOME}\\bin;${env.PATH}"
    MAVEN_HOME = tool name: 'Maven3', type: 'hudson.tasks.Maven$MavenInstallation'
    M2_HOME = "${MAVEN_HOME}"
    // 🔴 Eliminado: DEPLOY_HOST = credentials('deploy-host')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Test') {
      steps {
        bat """
          "%M2_HOME%\\bin\\mvn" -B -Dmaven.test.failure.ignore=false clean verify
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
        bat """
          "%M2_HOME%\\bin\\mvn" -B package -DskipTests
        """
      }
      post {
        success {
          archiveArtifacts artifacts: 'target\\*.jar', fingerprint: true
        }
      }
    }

    stage('Deploy to Staging') {
      when { expression { params.ENV == 'staging' } }
      steps {
        // Requiere: plugin SSH Agent + credencial "ssh-deploy-key" (SSH Username with private key)
        // y que el agente Windows tenga OpenSSH Client (ssh/scp) en PATH.
        sshagent(credentials: ['ssh-deploy-key']) {
          script {
            def host = params.DEPLOY_TARGET
            // Descubre el .jar
            bat '''
              for /f %%A in ('dir /b target\\*.jar') do set APP_JAR=target\\%%A
              echo Subiendo: %APP_JAR%
            '''
            // Sube y despliega
            bat "scp -o StrictHostKeyChecking=no %APP_JAR% " + host + ":/opt/apps/myapp/app.jar"
            bat "ssh -o StrictHostKeyChecking=no " + host + " \"sudo systemctl stop myapp || true && sudo mv /opt/apps/myapp/app.jar /opt/apps/myapp/current.jar || true && sudo systemctl start myapp && systemctl status myapp --no-pager -l\""
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
