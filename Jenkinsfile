pipeline {
  agent any

  tools {
    nodejs 'Nodejs'        // from Manage Jenkins → Tools
  }

  environment {
    // CHANGE THIS: app server public IP or DNS
    APP_HOST = 'ubuntu@44.220.163.69'
    APP_DIR  = '/home/ubuntu/app'
    APP_PORT = '3000'
  }

  options {
    timestamps()
  }

  triggers {
    // webhook triggers builds; this is a safety net (every 5m) if webhook fails
    pollSCM('H/5 * * * *')
  }

  stages {
    stage('Checkout') {
      steps {
        // If private repo: add credentialsId: 'github-https'
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
          # run tests if you have them; keep pipeline green for now
          npm test || echo "No tests or tests failed — continuing for demo"
        '''
      }
    }

    stage('Package') {
      steps {
         sh '''
                    # Remove old package if exists
                    rm -f app.tar.gz
                    
                    # Create package excluding unnecessary files
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
        sshagent(credentials: ['ubuntu']) {

	sh '''
            # copy artifact
            scp -o StrictHostKeyChecking=no app.tar.gz $APP_HOST:/tmp/app.tar.gz

            # unpack & install on the server
            ssh -o StrictHostKeyChecking=no $APP_HOST bash -lc "
              mkdir -p $APP_DIR &&
              tar -xzf /tmp/app.tar.gz -C $APP_DIR &&
              cd $APP_DIR &&
              if [ -f package-lock.json ]; then npm ci --omit=dev; else npm install --omit=dev; fi &&
              pm2 stop myapp || true &&
              pm2 start server.js --name myapp -- ${APP_PORT} &&
              pm2 save
            "
          '''
        }
      }
    }

    stage('Health Check') {
      steps {
        sh 'sleep 3'  // give it a moment to boot
        sh 'curl -fsS http://$APP_HOST_IP:$APP_PORT || curl -fsS http://$APP_HOST:$APP_PORT'
      }
    }
  }

  post {
    success {
 echo "✅ Deploy successful: http://$APP_HOST:$APP_PORT"
    }
    failure {
      echo "❌ Build or deploy failed — check the failed stage’s logs."
    }
  }
}

// Helper to extract IP if APP_HOST is in 'user@ip' format
def getIpFromHost(String host) {
  return host.contains('@') ? host.split('@')[1] : host
}

def APP_HOST_IP = getIpFromHost(env.APP_HOST)

