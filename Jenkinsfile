pipeline {
  agent any
  environment {
    APP_HOST = "3.108.239.215"   // <-- replace with your App EC2 public IP
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Build with Maven') {
      steps {
        sh 'mvn -B -DskipTests clean package'
      }
    }
    stage('Deploy with Ansible') {
      steps {
        sh '''
        set -eux

        # Create Ansible inventory
        cat > inventory.ini <<EOF
        [app]
        app1 ansible_host=${APP_HOST} ansible_user=ubuntu ansible_ssh_private_key_file=/var/lib/jenkins/.ssh/app.pem
        EOF

        # Create Ansible playbook
        cat > deploy.yml <<'EOF'
        - hosts: app
          become: yes
          tasks:
            - name: Install Java + Tomcat
              apt:
                name:
                  - openjdk-17-jdk
                  - tomcat9
                  - tomcat9-admin
                state: present
                update_cache: yes

            - name: Set JAVA_HOME for Tomcat
              lineinfile:
                path: /etc/default/tomcat9
                regexp: '^JAVA_HOME='
                line: 'JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64'
                create: yes

            - name: Copy WAR as ROOT.war
              copy:
                src: target/hello-web-1.0-SNAPSHOT.war
                dest: /var/lib/tomcat9/webapps/ROOT.war
                mode: '0644'

            - name: Restart Tomcat
              service:
                name: tomcat9
                state: restarted
        EOF

        # Run Ansible playbook
        ansible-playbook -i inventory.ini deploy.yml -vvv
        '''
      }
    }
  }
  post {
    success {
      echo "✅ Deployment successful! Access the app at: http://${APP_HOST}:8080/"
    }
    failure {
      echo "❌ Deployment failed! Check Jenkins console logs."
    }
  }
}
