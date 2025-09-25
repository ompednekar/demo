pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    ansiColor('xterm')
  }

  parameters {
    choice(name: 'ENV', choices: ['dev','test','prod'], description: 'Target environment')
    string(name: 'REGION', defaultValue: 'us-east-1', description: 'CloudHub 2.0 region')
    string(name: 'WORKER_TYPE', defaultValue: 'CH2.medium', description: 'Worker type')
    string(name: 'REPLICAS', defaultValue: '1', description: 'Replica count')
    string(name: 'RUNTIME_VERSION', defaultValue: '4.6.6', description: 'Mule runtime version')
    string(name: 'APP_NAME', defaultValue: 'my-mule-app', description: 'Application name')
  }

  environment {
    ANYPOINT_HOST = 'anypoint.mulesoft.com'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build + Test') {
      steps {
        sh 'mvn -B -U clean verify'
      }
      post {
        success {
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
          junit '**/surefire-reports/*.xml'
          junit '**/munit-reports/*.xml'
        }
      }
    }

    stage('Install Anypoint CLI') {
      steps {
        sh '''
          if ! command -v anypoint-cli-v4 >/dev/null 2>&1; then
            npm install -g anypoint-cli-v4@latest
          fi
          anypoint-cli-v4 --version
        '''
      }
    }

    stage('Login (Connected App)') {
      environment {
        ANYPOINT_CLIENT_ID     = credentials('ANYPOINT_CLIENT_ID')
        ANYPOINT_CLIENT_SECRET = credentials('ANYPOINT_CLIENT_SECRET')
        ANYPOINT_ORG_ID        = credentials('ANYPOINT_ORG_ID')
        ANYPOINT_ENV_ID        = credentials("ANYPOINT_ENV_ID_${ENV}")
      }
      steps {
        sh '''
          anypoint-cli-v4 login \
            --clientId "$ANYPOINT_CLIENT_ID" \
            --clientSecret "$ANYPOINT_CLIENT_SECRET" \
            --organization "$ANYPOINT_ORG_ID" \
            --environment "$ANYPOINT_ENV_ID" \
            --host "$ANYPOINT_HOST"
        '''
      }
    }

    stage('Deploy to CloudHub 2.0') {
      environment {
        ANYPOINT_ORG_ID = credentials('ANYPOINT_ORG_ID')
        ANYPOINT_ENV_ID = credentials("ANYPOINT_ENV_ID_${ENV}")
      }
      steps {
        sh '''
          set -e
          APP_JAR=$(ls -1 target/*.jar | head -n 1)

          # Create environment-specific properties
          cat > app-props.json <<'EOF'
          {
            "mule.env": "${ENV}",
            "anypoint.platform.config.analytics.agent.enabled": "true",
            "http.listener.host": "0.0.0.0",
            "http.listener.port": "8081"
          }
          EOF

          # Deploy to CloudHub 2.0
          anypoint-cli-v4 runtime-mgr cloudhub-2 applications deploy "$APP_NAME" "$APP_JAR" \
            --runtimeVersion "$RUNTIME_VERSION" \
            --region "$REGION" \
            --workers "$REPLICAS" \
            --workerType "$WORKER_TYPE" \
            --properties @app-props.json \
            --enableLogging true \
            --logLevel INFO \
            --organization "$ANYPOINT_ORG_ID" \
            --environment "$ANYPOINT_ENV_ID" \
            --wait
        '''
      }
    }

    stage('Post-Deploy Verification') {
      steps {
        sh '''
          # Verify deployment status
          STATUS=$(anypoint-cli-v4 runtime-mgr cloudhub-2 applications describe "$APP_NAME" --json | jq -r '.status')
          test "$STATUS" = "RUNNING"
          echo "✅ Application deployed successfully and is RUNNING"
        '''
      }
    }
  }

  post {
    always {
      sh 'anypoint-cli-v4 logout || true'
    }
  }
}
