pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    FE_DIR     = "front-end/webapp"
    BE_DIR     = "back-end/api"
    INFRA_DIR  = "infra"

    FE_IMAGE   = "myapp-frontend:local"
    BE_IMAGE   = "myapp-backend:local"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        // utile per i diff su PR: assicuriamoci di avere origin/master
        sh 'git fetch --no-tags --prune origin +refs/heads/*:refs/remotes/origin/*'
      }
    }

    stage('Detect changes') {
      steps {
        script {
          boolean isPR = (env.CHANGE_ID != null && env.CHANGE_ID.trim() != "")
          boolean isMaster = (env.BRANCH_NAME == "master")

          // Calcolo files cambiati:
          // - PR: diff tra merge-base con origin/master e HEAD
          // - Push: diff HEAD~1..HEAD (se esiste), altrimenti build everything
          List<String> files = []

          if (isPR) {
            def base = sh(script: "git merge-base HEAD origin/master", returnStdout: true).trim()
            def changed = sh(script: "git diff --name-only ${base}..HEAD", returnStdout: true).trim()
            files = changed ? (changed.split("\n") as List) : []
            echo "PR detected (CHANGE_ID=${env.CHANGE_ID}). Diff base=${base}"
            echo "Changed files:\n${changed}"
          } else {
            def canDiff = (sh(script: 'git rev-parse --verify HEAD~1 >/dev/null 2>&1; echo $?', returnStdout: true).trim() == "0")
            if (canDiff) {
              def changed = sh(script: "git diff --name-only HEAD~1..HEAD", returnStdout: true).trim()
              files = changed ? (changed.split("\n") as List) : []
              echo "Push build. Changed files:\n${changed}"
            } else {
              echo "No previous commit to diff against. Building everything."
              files = []
            }
          }

          // Se non riusciamo a determinare i file (es. primo build), buildiamo tutto
          boolean buildAll = files.isEmpty()

          boolean feChanged = buildAll || files.any { it.startsWith("front-end/") }
          boolean beChanged = buildAll || files.any { it.startsWith("back-end/") }
          boolean infraChanged = buildAll || files.any { it.startsWith("infra/") }
          boolean jenkinsfileChanged = buildAll || files.any { it == "Jenkinsfile" }

          // Se cambia Jenkinsfile o infra, ha senso ricostruire tutto (validazione end-to-end)
          if (jenkinsfileChanged || infraChanged) {
            feChanged = true
            beChanged = true
          }

          env.BUILD_FE = feChanged.toString()
          env.BUILD_BE = beChanged.toString()

          // Deploy automatico SOLO su master, NO PR
          env.DEPLOY_LOCAL = (isMaster && !isPR && (feChanged || beChanged || infraChanged || jenkinsfileChanged)).toString()

          echo "BUILD_FE=${env.BUILD_FE} BUILD_BE=${env.BUILD_BE} DEPLOY_LOCAL=${env.DEPLOY_LOCAL} | branch=${env.BRANCH_NAME} PR=${isPR}"
        }
      }
    }

    stage('Backend - Test & Package (Maven container)') {
      when { environment name: 'BUILD_BE', value: 'true' }
      agent {
        docker {
          image 'maven:3.9-eclipse-temurin-17'
          reuseNode true
        }
      }
      steps {
        dir("${BE_DIR}") {
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
          sh 'npm ci'
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
          // Usa il socket dell'host Docker Desktop
          args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
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

    stage('Deploy local (docker-compose)') {
      when { environment name: 'DEPLOY_LOCAL', value: 'true' }
      agent {
        docker {
          image 'docker:27-cli'
          reuseNode true
          args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        dir("${INFRA_DIR}") {
        if (env.BUILD_FE == "true" && env.BUILD_BE != "true") {
          sh 'docker-compose up -d --no-deps frontend'
        } else if (env.BUILD_BE == "true" && env.BUILD_FE != "true") {
          sh 'docker-compose up -d --no-deps backend'
        } else {
          sh 'docker-compose up -d'
        }
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