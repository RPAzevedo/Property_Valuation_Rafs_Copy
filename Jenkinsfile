pipeline {
  agent any

  environment {
    STAGING_URL    = 'https://houses-staging.kmaster.app'
    PRODUCTION_URL = 'https://houses.kmaster.app'
    HEALTH_PATH    = '/_stcore/health'
  }

  options {
    disableConcurrentBuilds()
  }

  triggers {
    pollSCM('* * * * *')
  }

  stages {
    // Code Quality Check and Tests share the workspace .venv (reuseNode), so they must
    // use the same Python. A different one makes uv delete and rebuild the whole .venv.
    stage('Code Quality Check') {
      agent {
        docker {
          image 'ghcr.io/astral-sh/uv:python3.13-bookworm-slim'
          reuseNode true
        }
      }
      steps {
        sh '''
          uv sync --locked
          uv run ruff check .
          uv run ruff format --check .
        '''
      }
    }

    stage('Tests') {
      agent {
        docker {
          image 'ghcr.io/astral-sh/uv:python3.13-bookworm-slim'
          reuseNode true
        }
      }
      steps {
        sh '''
          uv sync --locked
          uv run pytest -v --cov=estimator --cov=app --cov=scripts --cov-report=xml
        '''
      }
    }

    // stage('Security Analysis') {
    //   steps {
    //     withSonarQubeEnv('SonarCloud') {
    //       sh "${tool 'sonar-scanner'}/bin/sonar-scanner"
    //     }
    //   }
    // }

    // Kamal requires each image to carry its own service label, so the staging and production
    // images are derived from one build with a label-only layer. Their filesystems are identical.
    stage('Build and push image') {
      when { branch 'main' }
      steps {
        withCredentials([usernamePassword(credentialsId: 'ghcr-token',
                                          usernameVariable: 'KAMAL_REGISTRY_USERNAME',
                                          passwordVariable: 'KAMAL_REGISTRY_PASSWORD')]) {
          sh '''
            echo "$KAMAL_REGISTRY_PASSWORD" | docker login ghcr.io -u "$KAMAL_REGISTRY_USERNAME" --password-stdin
            docker buildx build --builder default --platform linux/amd64 --load \
              --build-arg GIT_SHA="$GIT_COMMIT" \
              --build-arg GIT_COMMITTED_AT="$(git log -1 --format=%cI)" \
              -t "property_valuation:$GIT_COMMIT" .
            for svc in property_valuation property_valuation_staging; do
              echo "FROM property_valuation:$GIT_COMMIT" |
                docker buildx build --builder default --platform linux/amd64 --push \
                  --label service=$svc -t "ghcr.io/nicolas2003/$svc:$GIT_COMMIT" -
            done
          '''
        }
      }
      post {
        always {
          sh '''
            docker image rm "property_valuation:$GIT_COMMIT" \
              "ghcr.io/nicolas2003/property_valuation:$GIT_COMMIT" \
              "ghcr.io/nicolas2003/property_valuation_staging:$GIT_COMMIT" || true
          '''
        }
      }
    }

    stage('Deploy to Staging') {
      when { branch 'main' }
      steps {
        sshagent(credentials: ['droplet-ssh']) {
          withCredentials([usernamePassword(credentialsId: 'ghcr-token',
                                            usernameVariable: 'KAMAL_REGISTRY_USERNAME',
                                            passwordVariable: 'KAMAL_REGISTRY_PASSWORD')]) {
            sh 'kamal deploy -d staging --skip-push --version "$GIT_COMMIT"'
          }
        }
      }
    }

    stage('Smoke test Staging') {
      when { branch 'main' }
      steps {
        sh 'curl -fsS --retry 10 --retry-delay 3 --retry-all-errors "$STAGING_URL$HEALTH_PATH"'
      }
    }

    stage('Deploy to Production') {
      when { branch 'main' }
      steps {
        sshagent(credentials: ['droplet-ssh']) {
          withCredentials([usernamePassword(credentialsId: 'ghcr-token',
                                            usernameVariable: 'KAMAL_REGISTRY_USERNAME',
                                            passwordVariable: 'KAMAL_REGISTRY_PASSWORD')]) {
            sh 'kamal deploy --skip-push --version "$GIT_COMMIT"'
          }
        }
      }
    }

    stage('Smoke test Production') {
      when { branch 'main' }
      steps {
        sh 'curl -fsS --retry 10 --retry-delay 3 --retry-all-errors "$PRODUCTION_URL$HEALTH_PATH"'
      }
    }

    stage('Monitoring and Alerting') {
      when {
        branch 'main'
        beforeAgent true
      }
      agent {
        docker {
          image 'ghcr.io/astral-sh/uv:python3.13-bookworm-slim'
          reuseNode true
        }
      }
      environment {
        APP_HEALTH_URL  = "${PRODUCTION_URL}${HEALTH_PATH}"
        UPTIME_CHECK_ID = '9eb1ff0c-fc8b-464f-8f59-0ce9f1821faa'
      }
      steps {
        withCredentials([string(credentialsId: 'digital-ocean-monitoring-token', variable: 'DO_TOKEN')]) {
          sh 'python3 scripts/check_monitoring.py'
        }
      }
    }
  }
}
