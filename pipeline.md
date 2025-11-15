## `pipeline-2-Tier`
```
pipeline {
    agent any

    environment {
        // RDS details
        RDS_ENDPOINT = 'ferozdb.cliumscw44qs.ap-south-1.rds.amazonaws.com'
        DB_NAME      = 'LoginDB'
        DB_USER      = 'admin'
        DB_PASS      = 'feroz123'

        // Apache directory
        DEPLOY_DIR   = '/var/www/html'

        // Git Repo
        GIT_REPO     = 'https://github.com/Ferozkhan24/GIT-REPO-2-TIER.git'
    }

    stages {

        stage('Clone Repo') {
            steps {
                echo "📥 Cloning project..."
                git branch: 'main', url: "${GIT_REPO}"
            }
        }

        stage('Install Apache & PHP') {
            steps {
                sh '''
                    sudo apt update -y
                    sudo apt install -y apache2 php libapache2-mod-php php-mysql mysql-client unzip
                    sudo systemctl start apache2
                    sudo systemctl enable apache2
                '''
            }
        }

        stage('Deploy PHP Application') {
            steps {
                sh '''
                    echo "🧹 Cleaning old files..."
                    sudo rm -rf ${DEPLOY_DIR}/*

                    echo "📦 Deploying all project files..."
                    sudo cp -rf * ${DEPLOY_DIR}/

                    echo "🔧 Fixing permissions..."
                    sudo chown -R www-data:www-data ${DEPLOY_DIR}
                    sudo chmod -R 755 ${DEPLOY_DIR}

                    echo "🔄 Restarting Apache..."
                    sudo systemctl restart apache2
                '''
            }
        }

        stage('Configure RDS Database') {
            steps {
                sh '''
                    mysql -h ${RDS_ENDPOINT} -u ${DB_USER} -p${DB_PASS} -e "
                        CREATE DATABASE IF NOT EXISTS ${DB_NAME};
                        USE ${DB_NAME};
                        CREATE TABLE IF NOT EXISTS users (
                            id INT AUTO_INCREMENT PRIMARY KEY,
                            username VARCHAR(50) NOT NULL,
                            password VARCHAR(255) NOT NULL
                        );
                    "
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    echo "🌐 Checking Web Application..."

                    def response = sh(
                        script: "curl -s -o page.html -w '%{http_code}' http://localhost",
                        returnStdout: true
                    ).trim()

                    echo "----- PAGE OUTPUT -----"
                    sh "cat page.html"
                    echo "-----------------------"

                    if (response == "200") {
                        echo "✅ Application reachable (HTTP 200)"
                    } else {
                        error "❌ Deployment failed (HTTP ${response}) — check the HTML output above."
                    }

                    echo "🔍 Checking RDS Connection..."
                    sh '''
                        mysql -h ${RDS_ENDPOINT} -u ${DB_USER} -p${DB_PASS} -e "SHOW DATABASES;"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Deployment Successful!"
        }
        failure {
            echo "💥 Deployment failed. Check Jenkins console logs."
        }
    }
}
```
