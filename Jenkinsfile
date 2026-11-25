pipeline {
    agent any

    environment {
        IMAGE_LOCAL  = 'backend-app:latest'
        IMAGE_REMOTE = 'sanamss/backend-app:latest'
        NETWORK      = 'devops-net'
        DB_NAME      = 'test_db'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test Maven') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${IMAGE_LOCAL} .'
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'REG_USER',
                        passwordVariable: 'REG_PASS')]) {
                    sh '''
                        echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin
                        docker tag ${IMAGE_LOCAL} ${IMAGE_REMOTE}
                        docker push ${IMAGE_REMOTE}
                    '''
                }
            }
        }

        stage('Deploy MySQL') {
            steps {
                sh '''
                    docker network create ${NETWORK} || true
                    docker rm -f mysql || true
                    docker run -d --name mysql --network ${NETWORK} \
                        -e MYSQL_ROOT_PASSWORD=root \
                        -e MYSQL_DATABASE=${DB_NAME} \
                        -p 3306:3306 mysql:8
                    echo "Attente de MySQL..."
                    for i in $(seq 1 30); do
                        docker exec mysql mysqladmin ping -h localhost -proot --silent && break
                        sleep 3
                    done
                '''
            }
        }

        stage('Deploy Backend') {
            steps {
                sh '''
                    docker stop backend-app || true
                    docker rm backend-app || true
                    docker pull ${IMAGE_REMOTE}
                    docker run -d --name backend-app --network ${NETWORK} \
                        -p 8089:8080 \
                        -e SPRING_DATASOURCE_URL="jdbc:mysql://mysql:3306/${DB_NAME}?createDatabaseIfNotExist=true" \
                        -e SPRING_DATASOURCE_USERNAME=root \
                        -e SPRING_DATASOURCE_PASSWORD=root \
                        ${IMAGE_REMOTE}
                '''
            }
        }

        stage('Verification') {
            steps {
                sh '''
                    for i in $(seq 1 20); do
                        docker logs backend-app 2>&1 | grep -q "Started BackendApplication" && break
                        sleep 3
                    done
                    docker ps
                    docker logs backend-app
                '''
            }
        }
    }
}
