pipeline {
    agent any

    environment {
        SONAR_TOKEN     = credentials('sonar-token')
        SONAR_HOST_URL  = credentials('sonar-host-url')
        EC2_HOST        = credentials('ec2-host')
        DB_HOST         = credentials('db-host')
        DB_USERNAME     = credentials('db-username')
        DB_PASSWORD     = credentials('db-password')
        EMAIL_USERNAME  = credentials('email-username')
        EMAIL_PASSWORD  = credentials('email-password')
        EMAIL_TO        = credentials('email-to')
    }

    stages {
        stage('Checkout Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/RameshBodduCloud/three-tier-app-cicd-pipeline.git'
            }
        }

        stage('Set up Java') {
            steps {
                sh '''
                sudo yum install -y java-17-amazon-corretto maven git jq
                '''
            }
        }

        stage('Build and SonarQube Scan') {
            steps {
                dir('backend') {
                    sh '''
                    mvn clean verify sonar:sonar \
                        -DskipTests \
                        -Dsonar.projectKey=backend \
                        -Dsonar.projectName=backend \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                // Jenkins SonarQube plugin can be used here if installed
                echo 'SonarQube Quality Gate check would run here'
            }
        }

        stage('Prepare EC2') {
            steps {
                sshagent(['ec2-key']) {
                    sh '''
                    ssh -o StrictHostKeyChecking=no ec2-user@$EC2_HOST "sudo yum install -y git maven java-17-amazon-corretto jq"
                    '''
                }
            }
        }

        stage('Clone / Pull Repository on EC2') {
            steps {
                sshagent(['ec2-key']) {
                    sh '''
                    ssh -o StrictHostKeyChecking=no ec2-user@$EC2_HOST '
                        set -e
                        APP_DIR=/home/ec2-user/app
                        REPO_NAME=three-tier-app-cicd-pipeline
                        REPO_URL=https://github.com/RameshBodduCloud/three-tier-app-cicd-pipeline.git
                        mkdir -p $APP_DIR
                        cd $APP_DIR
                        if [ -d "$REPO_NAME/.git" ]; then
                          cd $REPO_NAME
                          git reset --hard
                          git pull origin main
                        else
                          git clone $REPO_URL
                          cd $REPO_NAME
                        fi
                    '
                    '''
                }
            }
        }

        stage('Build Maven Application on EC2') {
            steps {
                sshagent(['ec2-key']) {
                    sh '''
                    ssh ec2-user@$EC2_HOST "
                        cd /home/ec2-user/app/three-tier-app-cicd-pipeline/backend &&
                        mvn clean package -DskipTests -U
                    "
                    '''
                }
            }
        }

        stage('Stop Old Application') {
            steps {
                sshagent(['ec2-key']) {
                    sh '''
                    ssh ec2-user@$EC2_HOST "
                        PID=$(ps -ef | grep '[d]atastore-0.0.7.jar' | awk '{print $2}')
                        if [ -z \\"$PID\\" ]; then
                          echo \\"Application is not running.\\"
                          exit 0
                        fi
                        echo \\"Stopping PID: $PID\\"
                        kill -15 $PID
                        sleep 5
                        if ps -p $PID > /dev/null 2>&1; then
                          echo \\"Force killing PID: $PID\\"
                          kill -9 $PID
                        fi
                        echo \\"Application stopped.\\"
                    "
                    '''
                }
            }
        }

        stage('Start Application') {
            steps {
                sshagent(['ec2-key']) {
                    sh '''
                    ssh ec2-user@$EC2_HOST "
                        LOG_DIR=/var/log/app
                        sudo mkdir -p $LOG_DIR
                        sudo chown ec2-user:ec2-user $LOG_DIR
                        cd /home/ec2-user/app/three-tier-app-cicd-pipeline/backend/target
                        nohup env \
                          MYSQL_HOST=$DB_HOST \
                          MYSQL_PORT=3306 \
                          MYSQL_USERNAME=$DB_USERNAME \
                          MYSQL_PASSWORD=$DB_PASSWORD \
                          LOG_FILE_PATH=$LOG_DIR/datastore.log \
                          java -jar datastore-0.0.7.jar --server.port=8084 \
                          > $LOG_DIR/nohup.out 2>&1 &
                    "
                    '''
                }
            }
        }

        stage('Wait for App Startup') {
            steps {
                sshagent(['ec2-key']) {
                    sh 'ssh ec2-user@$EC2_HOST "sleep 20"'
                }
            }
        }

        stage('Verify Application') {
            steps {
                sshagent(['ec2-key']) {
                    sh '''
                    ssh ec2-user@$EC2_HOST "
                        for i in {1..6}; do
                          HEALTH=$(curl -s http://localhost:8084/actuator/health || echo \\"curl failed\\")
                          echo \\"Attempt $i: $HEALTH\\"
                          STATUS=$(echo \\"$HEALTH\\" | jq -r '.status' 2>/dev/null || echo \\"UNKNOWN\\")
                          if [[ \\"$STATUS\\" == \\"UP\\" ]]; then
                            echo \\"✅ Application is healthy!\\"
                            exit 0
                          fi
                          sleep 10
                        done
                        echo \\"❌ Application did not become healthy in time\\"
                        exit 1
                    "
                    '''
                }
            }
        }

        stage('Send Deployment Report') {
            steps {
                // Jenkins Email Extension plugin can be configured here
                echo "Email report would be sent to $EMAIL_TO"
            }
        }
    }
}
