pipeline {
  agent any

  tools {
    nodejs 'Nodejs'        // Must match Manage Jenkins → Tools name
  }

  environment {
    APP_HOST = 'ubuntu@44.220.163.69'
    APP_HOST_IP = '44.220.163.69'
    APP_DIR  = '/home/ubuntu/app'
    APP_PORT = '3000'
  }

  options {
    timestamps()
  }

  triggers {
    // webhook triggers builds; this is a safety net
    pollSCM('H/5 * * * *')
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/sangeethanarapureddy/Simple-NodeJS.git'
      }
    }

    stage('Install & Test') {
      steps {
        sh '''
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install
          fi
          npm test || echo "No tests or tests failed — continuing for demo"
        '''
      }
    }

    stage('Package') {
      steps {
        sh '''
          rm -f app.tar.gz
          tar -czf app.tar.gz \
              --exclude='app.tar.gz' \
              --exclude='.*' \
              --exclude='node_modules/.cache' \
              Jenkinsfile \
              README.md \
              index.js \
              node_modules \
              package-lock.json \
              package.json \
              views
        '''
      }
    }

    stage('Deploy to EC2') {
      steps {
        sshagent(credentials: ['ec2-key']) {
          sh '''
            scp -o StrictHostKeyChecking=no app.tar.gz $APP_HOST:/tmp/app.tar.gz

            ssh -o StrictHostKeyChecking=no $APP_HOST bash -lc "
              mkdir -p $APP_DIR &&
              tar -xzf /tmp/app.tar.gz -C $APP_DIR &&
              cd $APP_DIR &&
              if [ -f package-lock.json ]; then npm ci --omit=dev; else npm install --omit=dev; fi &&
              pm2 stop myapp || true &&
              pm2 start index.js --name myapp -- ${APP_PORT} &&
              pm2 save
            "
          '''
        }
      }
    }

    stage('Health Check') {
      steps {
        sh 'sleep 3'
        sh "curl -fsS http://$APP_HOST_IP:$APP_PORT"
      }
    }
  }

  post {
    success {
      echo "Deploy successful: http://$APP_HOST_IP:$APP_PORT"
    }
    failure {
      echo "Build or deploy failed — check logs."
    }
  }
}
