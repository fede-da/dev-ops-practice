pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    FE_DIR   = "front-end/webapp"
    BE_DIR   = "back-end/api"
    INFRA_DIR= "infra"

    FE_IMAGE = "myapp-frontend:local"
    BE_IMAGE = "myapp-backend:local"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Detect changes') {
      steps {
        script {
          // Multibranch: quando è PR, Jenkins spesso espone CHANGE_TARGET / CHANGE_BRANCH
          // Per semplicità: confrontiamo con il commit precedente se disponibile
          def diffBase = sh(script: "git rev-parse HEAD~1 2>/dev/null || echo ''", returnStdout: true).trim()
          def changed = (diffBase)
            ? sh(script: "git diff --name-only ${diffBase}..HEAD", returnStdout: true).trim().split("\n") as List
            : []

          def feChanged = changed.any { it.startsWith("front-end/") }
          def beChanged = changed.any { it.startsWith("back-end/") }
          def infraChanged = changed.any { it.startsWith("infra/") }

          // Se non riusciamo a calcolare diff (primo build), buildiamo tutto
          if (changed.isEmpty()) {
            feChanged = true; beChanged = true; infraChanged = true
          }

          env.BUILD_FE = feChanged.toString()
          env.BUILD_BE = beChanged.toString()

          // Deploy SOLO su main (e quando qualcosa è cambiato)
          env.DEPLOY_LOCAL = (env.BRANCH_NAME == "main" && (feChanged || beChanged || infraChanged)).toString()

          echo "Changed files: ${changed}"
          echo "BUILD_FE=${env.BUILD_FE} BUILD_BE=${env.BUILD_BE} DEPLOY_LOCAL=${env.DEPLOY_LOCAL} (branch=${env.BRANCH_NAME})"
        }
      }
    }

    stage('Backend - Test/Package & Docker') {
      when { environment name: 'BUILD_BE', value: 'true' }
      steps {
        dir("${BE_DIR}") {
          sh './mvnw -B clean test package'
          sh "docker build -t ${BE_IMAGE} ."
        }
      }
    }

    stage('Frontend - Build & Docker') {
      when { environment name: 'BUILD_FE', value: 'true' }
      steps {
        dir("${FE_DIR}") {
          sh 'npm ci'
          sh 'npm run build'
          sh "docker build -t ${FE_IMAGE} ."
        }
      }
    }

    stage('Deploy local (docker compose)') {
      when { environment name: 'DEPLOY_LOCAL', value: 'true' }
      steps {
        dir("${INFRA_DIR}") {
          sh 'docker compose up -d --force-recreate'
          sh 'docker compose ps'
        }
      }
    }
  }

  post {
    always {
      echo "Done. Branch: ${env.BRANCH_NAME}"
    }
  }
}