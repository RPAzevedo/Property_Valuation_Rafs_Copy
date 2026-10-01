// ghcr-token is a username/password credential, but only its token is used. The registry
// username and owner are public and live in config/deploy.yml.
def withGhcrToken(Closure body) {
  withCredentials([usernamePassword(credentialsId: 'ghcr-token',
                                    usernameVariable: 'GHCR_TOKEN_USERNAME',
                                    passwordVariable: 'KAMAL_REGISTRY_PASSWORD')]) {
    body()
  }
}

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
        withGhcrToken {
          sh '''
            kamal registry login --skip-remote
            docker buildx build --builder default --platform linux/amd64 --load \
              --build-arg GIT_SHA="$GIT_COMMIT" \
              --build-arg GIT_COMMITTED_AT="$(git log -1 --format=%cI)" \
              -t "property_valuation:$GIT_COMMIT" .
            # Image and service names come from config/deploy*.yml, so the registry owner is set there.
            for dest in "" "-d staging"; do
              config=$(kamal config $dest --version "$GIT_COMMIT")
              image=$(echo "$config" | awk '/^:absolute_image:/ {print $2}')
              svc=$(echo "$config" | awk '/^:service_with_version:/ {print $2}')
              echo "FROM property_valuation:$GIT_COMMIT" |
                docker buildx build --builder default --platform linux/amd64 --push \
                  --label service="${svc%-$GIT_COMMIT}" -t "$image" -
            done
          '''
        }
      }
      post {
        always {
          sh '''
            docker image rm "property_valuation:$GIT_COMMIT" \
              $(for dest in "" "-d staging"; do
                  kamal config $dest --version "$GIT_COMMIT" | awk '/^:absolute_image:/ {print $2}'
                done) || true
          '''
        }
      }
    }

    stage('Deploy to Staging') {
      when { branch 'main' }
      steps {
        sshagent(credentials: ['droplet-ssh']) {
          withGhcrToken {
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
          withGhcrToken {
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
