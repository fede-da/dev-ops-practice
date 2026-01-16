pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    FE_DIR    = "front-end/webapp"
    BE_DIR    = "back-end/api"
    INFRA_DIR = "infra"

    FE_IMAGE  = "myapp-frontend:local"
    BE_IMAGE  = "myapp-backend:local"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'git --version'
      }
    }

    stage('Detect changes') {
      steps {
        script {
          // Default: build everything if we can't diff
          boolean feChanged = true
          boolean beChanged = true
          boolean infraChanged = true

          // If repo has at least 2 commits, detect changed paths vs previous commit
          def canDiff = (sh(script: 'git rev-parse --verify HEAD~1 >/dev/null 2>&1; echo $?', returnStdout: true).trim() == "0")
          if (canDiff) {
            def changed = sh(script: "git diff --name-only HEAD~1..HEAD", returnStdout: true).trim()
            def files = changed ? changed.split("\n") : []

            feChanged = files.any { it.startsWith("front-end/") }
            beChanged = files.any { it.startsWith("back-end/") }
            infraChanged = files.any { it.startsWith("infra/") }

            echo "Changed files:\n${changed}"
          } else {
            echo "No previous commit to diff against. Building everything."
          }

          env.BUILD_FE = feChanged.toString()
          env.BUILD_BE = beChanged.toString()

          // Deploy only on master AND only when it's not a PR build
          def isMaster = (env.BRANCH_NAME == "master")
          def isPR = (env.CHANGE_ID != null && env.CHANGE_ID.trim() != "")
          env.DEPLOY_LOCAL = (isMaster && !isPR && (feChanged || beChanged || infraChanged)).toString()

          echo "BUILD_FE=${env.BUILD_FE} BUILD_BE=${env.BUILD_BE} DEPLOY_LOCAL=${env.DEPLOY_LOCAL} | branch=${env.BRANCH_NAME} PR=${isPR}"
        }
      }
    }

    stage('Backend - Test & Package (Maven container)') {
      when { environment name: 'BUILD_BE', value: 'true' }
      agent {
        docker {
          image 'maven:3.9-eclipse-temurin-17'
          // reuseNode monta la workspace dentro il container
          reuseNode true
        }
      }
      steps {
        dir("${BE_DIR}") {
          sh 'java -version'
          sh 'mvn -v'
          // build vera
          sh 'mvn -B clean test package'
        }
      }
    }

    stage('Frontend - Build (Node container)') {
      when { environment name: 'BUILD_FE', value: 'true' }
      agent {
        docker {
          image 'node:20-alpine'
          reuseNode true
        }
      }
      steps {
        dir("${FE_DIR}") {
          sh 'node -v'
          sh 'npm -v'
          sh 'npm ci'
          // Per ora: niente unit test headless in CI (lo sistemiamo nello step successivo con Playwright o Chrome container)
          sh 'npm run build'
        }
      }
    }

    stage('Docker Build Images (Docker CLI container)') {
      when {
        anyOf {
          environment name: 'BUILD_FE', value: 'true'
          environment name: 'BUILD_BE', value: 'true'
        }
      }
      agent {
        docker {
          image 'docker:27-cli'
          reuseNode true
          // socket necessario per docker build
          args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        sh 'docker version'

        script {
          if (env.BUILD_BE == "true") {
            dir("${BE_DIR}") {
              sh "docker build -t ${BE_IMAGE} ."
            }
          }
          if (env.BUILD_FE == "true") {
            dir("${FE_DIR}") {
              sh "docker build -t ${FE_IMAGE} ."
            }
          }
        }
      }
    }

    stage('Deploy local (docker compose)') {
      when { environment name: 'DEPLOY_LOCAL', value: 'true' }
      agent {
        docker {
          image 'docker:27-cli'
          reuseNode true
          args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        dir("${INFRA_DIR}") {
          // docker compose v2 dovrebbe esserci; se non c'è, ti do workaround
          sh 'docker compose version'
          sh 'docker compose up -d --force-recreate'
          sh 'docker compose ps'
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline completed. Branch=${env.BRANCH_NAME}"
    }
  }
}