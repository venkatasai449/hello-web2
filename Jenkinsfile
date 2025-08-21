pipeline {
  agent any
  environment {
    // 🔹 Replace with your App EC2 Public IP
    APP_HOST = "3.108.239.215"
  }

  stages {
    stage('Checkout') {
      steps {
        // Explicit git checkout works in both Pipeline script and Pipeline from SCM
        git branch: 'main', url: 'https://github.com/venkatasai449/hello-web.git'
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

        # 🔹 Create inventory file for Ansible
        cat > inventory.ini <<EOF
        [app]
        app1 ansible_host=${APP_HOST} ansible_user=ubuntu ansible_ssh_private_key_file=/var/lib/jenkins/.ssh/app.pem
        EOF

        # 🔹 Create Ansible playbook
        cat > deploy.yml <<'EOF'
        - hosts: app
          become: yes
          tasks:
            - name: Install Tomcat 9
              apt:
                name:
                  - tomcat9
                  - tomcat9-admin
                state: present
                update_cache: yes

            - name: Copy WAR as ROOT.war
              copy:
                src: target/hello.war
                dest: /var/lib/tomcat9/webapps/ROOT.war
                mode: '0644'

            - name: Restart Tomcat
              service:
                name: tomcat9
                state: restarted
        EOF

        # 🔹 Run Ansible playbook
        ansible-playbook -i inventory.ini deploy.yml
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Deployment successful! Access the app at: http://${APP_HOST}:8080/"
    }
    failure {
      echo "❌ Build or deployment failed. Check console logs."
    }
  }
}
