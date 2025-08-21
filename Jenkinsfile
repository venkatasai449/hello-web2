pipeline {
  agent any
  environment {
    APP_HOST = "3.108.239.215"
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
      }
    }

    stage('Deploy with Ansible') {
      steps {
        sh '''
        set -eux

        # Create inventory
        cat > inventory.ini <<EOF
        [app]
        app1 ansible_host=${APP_HOST} ansible_user=ubuntu ansible_ssh_private_key_file=/var/lib/jenkins/.ssh/app.pem
        EOF

        # Create playbook
        cat > deploy.yml <<EOF
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
                src: target/hello-web-1.0-SNAPSHOT.war
                dest: /var/lib/tomcat9/webapps/ROOT.war
                mode: '0644'

            - name: Restart Tomcat
              service:
                name: tomcat9
                state: restarted
        EOF

        # Run playbook
        ansible-playbook -i inventory.ini deploy.yml
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Deployment successful! Access: http://${APP_HOST}:8080/"
    }
    failure {
      echo "❌ Deployment failed. Check console logs."
    }
  }
}
