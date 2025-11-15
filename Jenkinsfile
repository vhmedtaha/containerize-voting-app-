pipeline {
  agent any

  parameters {
    booleanParam(name: 'PUSH_IMAGES', defaultValue: false, description: 'If true, push built images to the provided registry')
    string(name: 'REGISTRY', defaultValue: '', description: 'Docker registry prefix (e.g. ghcr.io/username or docker.io/username)')
    string(name: 'REGISTRY_CREDENTIALS_ID', defaultValue: '', description: 'Jenkins credentials id for registry login (username/password)')
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Optional image tag. If empty, Jenkins build id will be used')
    booleanParam(name: 'RUN_CONTAINERS', defaultValue: true, description: 'If true, run `docker compose up` to start the stack after building')
    booleanParam(name: 'RUN_SEED', defaultValue: false, description: 'If true, run the `seed` service after stack is healthy')
    string(name: 'HEALTH_MAX_ATTEMPTS', defaultValue: '60', description: 'Number of attempts to wait for service health (5s per attempt)')
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
          if (!dockerPath) {
            error "Docker CLI not found on this node. Ensure Docker is installed or run Jenkins with the Docker socket mounted."
          }

          // detect docker compose command (docker compose vs docker-compose)
          def composeCmd = sh(script: 'if docker compose version >/dev/null 2>&1; then echo "docker compose"; elif docker-compose version >/dev/null 2>&1; then echo "docker-compose"; else echo ""; fi', returnStdout: true).trim()
          if (!composeCmd) {
            error "docker compose (or docker-compose) not found on this node. Install Docker Compose or use a Docker-enabled agent."
          }
          env.COMPOSE_CMD = composeCmd
          echo "Using compose command: ${env.COMPOSE_CMD}"
        }
      }
    }

    stage('Build images') {
      steps {
        script {
          def services = ['vote','result','worker','seed-data']
          for (s in services) {
            def prefix = params.REGISTRY?.trim() ? params.REGISTRY.trim() + '/' : ''
            def imageName = "${prefix}${env.JOB_NAME}-${s}:${env.IMAGE_TAG}"
            echo "Building ${imageName} from ./${s}"
            sh "docker build -t ${imageName} ${s}"
          }
        }
      }
    }

    stage('Push images') {
      when { expression { return params.PUSH_IMAGES } }
      steps {
        script {
          if (!params.REGISTRY?.trim()) { error 'PUSH_IMAGES is true but REGISTRY is empty — provide a registry to push to.' }
          if (!params.REGISTRY_CREDENTIALS_ID?.trim()) { error 'REGISTRY_CREDENTIALS_ID is required to push images securely.' }

          withCredentials([usernamePassword(credentialsId: params.REGISTRY_CREDENTIALS_ID, usernameVariable: 'REG_USER', passwordVariable: 'REG_PSW')]) {
            sh "echo \"\\$REG_PSW\" | docker login ${params.REGISTRY} -u \"\\$REG_USER\" --password-stdin"

            def services = ['vote','result','worker','seed-data']
            for (s in services) {
              def imageName = "${params.REGISTRY}/${env.JOB_NAME}-${s}:${env.IMAGE_TAG}"
              echo "Pushing ${imageName}"
              sh "docker push ${imageName}"
            }
          }
        }
      }
    }

    stage('Deploy (docker compose)') {
      when { expression { return params.RUN_CONTAINERS } }
      steps {
        script {
          echo "Starting docker compose stack"
          sh "${env.COMPOSE_CMD} up -d --build"

          // wait for health status of redis and postgres (container names are set in compose)
          def servicesToWait = ['devops-redis','devops-postgres']
          def maxAttempts = params.HEALTH_MAX_ATTEMPTS.toInteger()
          for (svc in servicesToWait) {
            echo "Waiting for ${svc} to become healthy (max ${maxAttempts} attempts)"
            def attempt = 0
            def healthy = false
            while (attempt < maxAttempts) {
              def status = sh(script: "docker inspect -f '{{.State.Health.Status}}' ${svc} 2>/dev/null || echo 'no'", returnStdout: true).trim()
              echo "${svc} status: ${status}"
              if (status == 'healthy') { healthy = true; break }
              attempt++
              sleep time: 5, unit: 'SECONDS'
            }
            if (!healthy) { error "${svc} did not become healthy after ${maxAttempts} attempts" }
            echo "${svc} is healthy"
          }
        }
      }
    }

    stage('Seed data (optional)') {
      when { allOf { expression { return params.RUN_CONTAINERS }; expression { return params.RUN_SEED } } }
      steps {
        script {
          echo "Running seed job to populate test votes"
          sh "${env.COMPOSE_CMD} run --rm seed"
        }
      }
    }

    stage('Smoke & Logs') {
      steps {
        script {
          echo 'Listing containers and recent logs'
          sh "${env.COMPOSE_CMD} ps"
          sh "${env.COMPOSE_CMD} logs --no-color --tail=200"
        }
      }
    }

    stage('Archive build outputs') {
      steps { archiveArtifacts artifacts: 'docker-compose.yml, **/Dockerfile', allowEmptyArchive: true }
    }
  }

  post {
    success { echo "Pipeline finished successfully. Images built with tag: ${env.IMAGE_TAG}" }
    failure { echo "Pipeline failed. Check console output for errors." }
  }
}
