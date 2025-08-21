pipeline {
  agent any
  environment {
    APP_HOST = "3.108.239.215"         // <-- your App EC2 public IP
    ANSIBLE_HOST_KEY_CHECKING = 'False'
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/venkatasai449/hello-web.git'
      }
    }

    stage('Build with Maven') {
      steps {
        sh 'mvn -B -DskipTests clean package'
        sh 'ls -l target/'
      }
    }

    stage('Deploy with Ansible') {
      steps {
        sh '''
        set -eux

        # Write inventory
        cat > inventory.ini <<EOF
        [app]
        app1 ansible_host=${APP_HOST} ansible_user=ubuntu ansible_ssh_private_key_file=/var/lib/jenkins/.ssh/app.pem
        EOF

        # Write playbook (NO quoted heredoc — so the file is actually created)
        cat > deploy.yml <<EOF
        - hosts: app
          become: yes
          vars:
            java_home: /usr/lib/jvm/java-17-openjdk-amd64
          tasks:
            - name: Update apt cache
              apt:
                update_cache: yes

            - name: Install Java 17 + Tomcat9
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

            - name: Reload systemd
              systemd:
                daemon_reload: yes

            - name: Start & enable Tomcat9
              systemd:
                name: tomcat9
                state: started
                enabled: yes

            - name: Wait for Tomcat :8080
              wait_for:
                port: 8080
                host: 127.0.0.1
                delay: 2
                timeout: 60

            - name: Remove exploded ROOT/ if exists
              file:
                path: /var/lib/tomcat9/webapps/ROOT
                state: absent

            - name: Copy WAR as ROOT.war
              copy:
                src: target/hello-web-1.0-SNAPSHOT.war
                dest: /var/lib/tomcat9/webapps/ROOT.war
                mode: '0644'

            - name: Restart Tomcat
              systemd:
                name: tomcat9
                state: restarted

            - name: Wait after restart
              wait_for:
                port: 8080
                host: 127.0.0.1
                delay: 2
                timeout: 60

            - name: HTTP check (localhost)
              uri:
                url: http://localhost:8080/
                status_code: 200
        EOF

        # Run
        ansible-playbook -i inventory.ini deploy.yml -vv
        '''
      }
    }

    stage('Smoke test from Jenkins box') {
      steps {
        sh 'curl -I http://${APP_HOST}:8080/ || (echo "App not reachable" && exit 1)'
      }
    }
  }

  post {
    success {
      echo "✅ Deployed: http://${APP_HOST}:8080/"
    }
    failure {
      echo "❌ Deployment failed. Check Ansible section in the log."
    }
  }
}
