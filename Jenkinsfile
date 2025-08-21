pipeline {
  agent any
  environment {
    APP_HOST = "3.108.239.215"  // <-- put your App EC2 public IP here
  }
  stages {
    stage('Checkout') {
      steps {
        // Jenkins will override this if you configure "Pipeline from SCM",
        // but keeping it here makes "Pipeline script" also work.
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

        # Inventory for Ansible
        cat > inventory.ini <<EOF
        [app]
        app1 ansible_host=${APP_HOST} ansible_user=ubuntu ansible_ssh_private_key_file=/var/lib/jenkins/.ssh/app.pem
        EOF

        # Playbook: install Tomcat9 and deploy WAR
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

        ansible-playbook -i inventory.ini deploy.yml
        '''
      }
    }
  }
  post {
    success {
      echo "App URL: http://${APP_HOST}:8080/"
    }
  }
}
