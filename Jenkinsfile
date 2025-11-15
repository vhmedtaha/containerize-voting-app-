pipeline {
  agent any

  parameters {
    booleanParam(name: 'PUSH_IMAGES', defaultValue: false, description: 'If true, push built images to the provided registry')
    string(name: 'REGISTRY', defaultValue: '', description: 'Docker registry prefix (e.g. ghcr.io/username or registry.hub.docker.com/username). Leave empty to skip registry prefix')
    string(name: 'REGISTRY_CREDENTIALS_ID', defaultValue: '', description: 'Jenkins credentials id (username/password) for registry login')
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Optional image tag. If empty, Jenkins build id will be used')
  }

  environment {
    IMAGE_TAG = "${params.IMAGE_TAG ?: env.BUILD_ID}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Pre-check') {
      steps {
        script {
          echo "Using image tag: ${env.IMAGE_TAG}"
          sh 'docker --version || true'
        }
      }
    }

    stage('Build images') {
      steps {
        script {
          def services = ['vote','result','worker','seed-data']
          for (s in services) {
            def imageName = (params.REGISTRY?.trim() ? params.REGISTRY + '/' : '') + "${env.JOB_NAME}-${s}:${env.IMAGE_TAG}"
            echo "Building ${imageName} from ./${s}"
            sh "docker build -t ${imageName} ${s}"
          }
        }
      }
    }

    stage('Push images') {
      when {
        expression { return params.PUSH_IMAGES }
      }
      steps {
        script {
          if (!params.REGISTRY?.trim()) {
            error 'PUSH_IMAGES is true but REGISTRY is empty — provide a registry to push to.'
          }

          if (!params.REGISTRY_CREDENTIALS_ID?.trim()) {
            error 'REGISTRY_CREDENTIALS_ID is required to push images securely.'
          }

          withCredentials([usernamePassword(credentialsId: params.REGISTRY_CREDENTIALS_ID, usernameVariable: 'REG_USER', passwordVariable: 'REG_PSW')]) {
            sh 'echo "$REG_PSW" | docker login ${params.REGISTRY} -u "$REG_USER" --password-stdin'

            def services = ['vote','result','worker','seed-data']
            for (s in services) {
              def imageName = params.REGISTRY + '/' + "${env.JOB_NAME}-${s}:${env.IMAGE_TAG}"
              echo "Pushing ${imageName}"
              sh "docker push ${imageName}"
            }
          }
        }
      }
    }

    stage('Archive build outputs') {
      steps {
        archiveArtifacts artifacts: 'docker-compose.yml, **/Dockerfile', allowEmptyArchive: true
      }
    }
  }

  post {
    success {
      echo "Pipeline finished successfully. Images built with tag: ${env.IMAGE_TAG}"
    }
    failure {
      echo "Pipeline failed. Check console output for errors."
    }
  }
}
