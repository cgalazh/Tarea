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
    // Ajusta los nombres a los Tools que tengas configurados
    JAVA_HOME = tool name: 'JDK21', type: 'hudson.model.JDK'
    PATH = "${JAVA_HOME}\\bin;${env.PATH}"
    MAVEN_HOME = tool name: 'Maven3', type: 'hudson.tasks.Maven$MavenInstallation'
    M2_HOME = "${MAVEN_HOME}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Test') {
      steps {
        script {
          if (fileExists('pom.xml')) {
            // Proyecto Maven
            bat "\"%M2_HOME%\\bin\\mvn\" -B -Dmaven.test.failure.ignore=false clean verify"
          } else {
            // Placeholder si NO hay Maven project
            echo "No se encontró pom.xml → se omite build Maven"
            bat 'echo Build placeholder OK'
          }
        }
      }
      post {
        always {
          // No falla aunque no existan reportes
          junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
        }
      }
    }

    stage('Package') {
      when { expression { fileExists('pom.xml') } }
      steps {
        bat "\"%M2_HOME%\\bin\\mvn\" -B package -DskipTests"
      }
      post {
        success {
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, allowEmptyArchive: true
        }
      }
    }

    /* ---------------------  DESPLEGUE COMENTADO  ---------------------
    stage('Deploy to Staging') {
      when { expression { params.ENV == 'staging' } }
      steps {
        echo 'Deploy desactivado por ahora'
      }
    }
    ------------------------------------------------------------------- */
  }

  post {
    success { echo "✅ Build OK: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
    failure { echo "❌ Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
  }
}
