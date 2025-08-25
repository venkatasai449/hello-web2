pipeline {
  agent any
  options { timestamps(); skipDefaultCheckout(true) }

  environment {
    APP_HOST = "54.198.173.58"                 // <-- change to your App EC2 public IP
    SSH_KEY  = "/var/lib/jenkins/.ssh/app.pem" // key Jenkins uses to SSH to App EC2
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/venkatasai449/hello-web.git'
        sh 'set -eux; echo "Workspace: $WORKSPACE"; ls -la'
      }
    }

    stage('Build with Maven') {
      steps {
        sh '''
          set -eux
          mvn -B -DskipTests clean package
          ls -l target/
        '''
      }
    }

    stage('Deploy to Tomcat (no Ansible)') {
      steps {
        sh '''
          set -eux

          # sanity: key present
          test -f "$SSH_KEY" || { echo "Missing SSH key at $SSH_KEY"; exit 1; }
          chmod 600 "$SSH_KEY" || true

          # avoid interactive host key prompts
          mkdir -p ~/.ssh
          ssh-keyscan -H "${APP_HOST}" >> ~/.ssh/known_hosts 2>/dev/null || true

          # 1) Prepare App EC2: install Java 17 + Tomcat9, set JAVA_HOME, enable & start
          ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no ubuntu@${APP_HOST} '
            set -eux
            sudo apt-get update -y
            sudo apt-get install -y openjdk-17-jdk tomcat9 tomcat9-admin
            sudo mkdir -p /etc/systemd/system/tomcat9.service.d
            echo "[Service]
Environment=JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64" | sudo tee /etc/systemd/system/tomcat9.service.d/override.conf >/dev/null
            sudo systemctl daemon-reload
            sudo systemctl enable tomcat9
            sudo systemctl restart tomcat9
          '

          # 2) Copy the WAR built by Maven to the server
          scp -i "$SSH_KEY" -o StrictHostKeyChecking=no target/hello-web-1.0-SNAPSHOT.war ubuntu@${APP_HOST}:/tmp/app.war

          # 3) Replace ROOT.war and restart Tomcat
          ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no ubuntu@${APP_HOST} '
            set -eux
            sudo rm -rf /var/lib/tomcat9/webapps/ROOT /var/lib/tomcat9/webapps/ROOT.war || true
            sudo mv /tmp/app.war /var/lib/tomcat9/webapps/ROOT.war
            sudo systemctl restart tomcat9
          '

          # 4) Smoke test from Jenkins
          sleep 5
          curl -I "http://${APP_HOST}:8080/" | head -n1
        '''
      }
    }
  }

  post {
    success { echo "✅ Deployment successful! Open: http://${APP_HOST}:8080/" }
    failure { echo "❌ Deployment failed. Check the first error in the Deploy stage." }
  }
}
