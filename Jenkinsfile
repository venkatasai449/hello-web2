pipeline {
  agent any
  options {
    timestamps()
    skipDefaultCheckout(true)
  }
  environment {
    APP_HOST = "3.108.239.215"                 // <-- your App EC2 public IP
    ANSIBLE_HOST_KEY_CHECKING = 'False'        // avoid first-time SSH prompt
    SSH_KEY = "/var/lib/jenkins/.ssh/app.pem"  // private key Jenkins uses to SSH to app
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/venkatasai449/hello-web.git'
        sh '''
          set -eux
          echo "Workspace: $WORKSPACE"
          ls -la
        '''
      }
    }

    stage('Build with Maven') {
      steps {
        sh '''
          set -eux
          mvn -B -DskipTests clean package
          echo "Built artifacts:"
          ls -l target/
        '''
      }
    }

    stage('Prepare Ansible Files') {
      steps {
        sh '''
          set -eux

          # Ensure SSH key exists & readable
          test -f "$SSH_KEY" || { echo "Missing SSH key at $SSH_KEY"; exit 1; }
          chmod 600 "$SSH_KEY" || true

          # Pre-trust the app host to avoid interactive prompts
          mkdir -p ~/.ssh
          ssh-keyscan -H ${APP_HOST} >> ~/.ssh/known_hosts 2>/dev/null || true

          # Inventory for Ansible
          cat > inventory.ini <<EOF
          [app]
          app1 ansible_host=${APP_HOST} ansible_user=ubuntu ansible_ssh_private_key_file=${SSH_KEY}
          EOF
          echo "----- inventory.ini -----"
          cat inventory.ini

          # Playbook: Install Java+Tomcat, set JAVA_HOME via systemd override, deploy WAR, health-check
          cat > deploy.yml <<EOF
          - hosts: app
            become: yes
            vars:
              java_home: /usr/lib/jvm/java-17-openjdk-amd64
            tasks:
              - name: Update apt cache
                apt:
                  update_cache: yes

              - name: Install Java 17 and Tomcat 9
                apt:
                  name:
                    - openjdk-17-jdk
                    - tomcat9
                    - tomcat9-admin
                  state: present

              - name: Ensure systemd override dir exists
                file:
                  path: /etc/systemd/system/tomcat9.service.d
                  state: directory
                  mode: '0755'

              - name: Set JAVA_HOME via systemd override
                copy:
                  dest: /etc/systemd/system/tomcat9.service.d/override.conf
                  mode: '0644'
                  content: |
                    [Service]
                    Environment="JAVA_HOME={{ java_home }}"

              - name: Reload systemd to pick up override
                systemd:
                  daemon_reload: yes

              - name: Ensure Tomcat is started and enabled
                systemd:
                  name: tomcat9
                  state: started
                  enabled: yes

              - name: Wait for Tomcat to listen on 8080 (first start)
                wait_for:
                  host: 127.0.0.1
                  port: 8080
                  delay: 2
                  timeout: 90
                  state: started

              - name: Remove exploded ROOT/ if present (avoid stale files)
                file:
                  path: /var/lib/tomcat9/webapps/ROOT
                  state: absent

              - name: Copy WAR to app as ROOT.war
                copy:
                  src: target/hello-web-1.0-SNAPSHOT.war
                  dest: /var/lib/tomcat9/webapps/ROOT.war
                  mode: '0644'

              - name: Restart Tomcat after deploy
                systemd:
                  name: tomcat9
                  state: restarted

              - name: Wait for Tomcat after restart
                wait_for:
                  host: 127.0.0.1
                  port: 8080
                  delay: 2
                  timeout: 90
                  state: started

              - name: HTTP check from app node (localhost)
                uri:
                  url: http://localhost:8080/
                  status_code: 200
          EOF

          echo "----- deploy.yml (first 120 lines) -----"
          sed -n '1,120p' deploy.yml
        '''
      }
    }

    stage('Preflight: Ansible ping') {
      steps {
        sh '''
          set -eux
          ansible -i inventory.ini app -m ping -vv || { echo "Ansible ping failed"; exit 1; }
        '''
      }
    }

    stage('Deploy with Ansible') {
      steps {
        sh '''
          set -eux
          ansible-playbook -i inventory.ini deploy.yml -vv
        '''
      }
    }

    stage('Smoke test from Jenkins') {
      steps {
        sh '''
          set -eux
          curl -I "http://${APP_HOST}:8080/" | head -n1
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Deployment successful! Open: http://${APP_HOST}:8080/"
    }
    failure {
      echo "❌ Deployment failed. Check the first 'fatal:' in Ansible logs above."
    }
  }
}

