pipeline {
  agent any

  parameters {
    booleanParam(name: 'RUN_CONTAINERS', defaultValue: true, description: 'Run the stack with docker compose')
    booleanParam(name: 'RUN_SEED', defaultValue: false, description: 'Run the seed service after stack is healthy')
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Optional image tag (defaults to BUILD_ID)')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Pre-check') {
      steps {
        script {
          env.IMAGE_TAG = params.IMAGE_TAG?.trim() ? params.IMAGE_TAG.trim() : env.BUILD_ID
          echo "Using image tag: ${env.IMAGE_TAG}"

          def dockerPath = sh(script: 'which docker || true', returnStdout: true).trim()
          if (!dockerPath) { error 'Docker CLI not found on this node. Ensure Docker is available (mount /var/run/docker.sock or run on a Docker-capable agent).' }

          def compose = sh(script: 'if docker compose version >/dev/null 2>&1; then echo "docker compose"; elif docker-compose version >/dev/null 2>&1; then echo "docker-compose"; else echo ""; fi', returnStdout: true).trim()
          if (!compose) { error 'docker compose (or docker-compose) not found on this node.' }
          env.COMPOSE_CMD = compose
          echo "Compose command: ${env.COMPOSE_CMD}"
        }
      }
    }

    stage('Start stack') {
      when { expression { return params.RUN_CONTAINERS } }
      steps {
        script {
          echo 'Bringing up stack (builds images and starts services)'
          sh "${env.COMPOSE_CMD} up -d --build"
          sleep time: 5, unit: 'SECONDS'
          sh "${env.COMPOSE_CMD} ps"
        }
      }
    }

    stage('Seed (optional)') {
      when { allOf { expression { return params.RUN_CONTAINERS }; expression { return params.RUN_SEED } } }
      steps {
        script {
          echo 'Running seed service to populate test data'
          sh "${env.COMPOSE_CMD} run --rm seed"
        }
      }
    }

    stage('Logs') {
      steps {
        script {
          echo 'Collecting recent logs for quick troubleshooting'
          sh "${env.COMPOSE_CMD} logs --no-color --tail=200 || true"
        }
      }
    }

  }

  post {
    success { echo 'Pipeline finished successfully.' }
    failure { echo 'Pipeline failed — check logs above.' }
  }
}
