pipeline {

  agent any

    environment {                                                   
        DOCKER_USERNAME = 'snezhok99'
        DOCKER_PASSWORD_ID = 'c6b31924-c8a4-4ab3-bfb6-6139260a7909'
        SWARM_STACK_NAME = 'app'
    }

stages {
        stage('Build Docker Images') {
            steps {
                script {
                    // Авторизация в Docker Hub
                    withCredentials([usernamePassword(credentialsId: env.DOCKER_PASSWORD_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }

                    // Сборка образа для PHP/Apache (web-server)
                    sh "docker build -f php.Dockerfile . -t ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-web:latest"
                    // Сборка образа для MySQL (db) - если вы используете свой образ MySQL
                    sh "docker build -f mysql.Dockerfile . -t ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-db:latest"
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
                    // Отправка образов в Docker Hub
                    sh "docker push ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-web:latest"
                    sh "docker push ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-db:latest" // Если вы используете свой образ MySQL
                }
            }
        }

        stage('Deploy to Docker Swarm') {
            steps {
                script {
                    // Проверка, что Docker Swarm инициализирован
                    // Этот шаг предполагает, что Jenkins агент имеет доступ к Docker Swarm API
                    // через монтирование docker.sock
                    sh "docker swarm init || true"
                    sh "docker stack deploy --with-registry-auth -c docker-compose.yaml ${SWARM_STACK_NAME}"
                }
            }
        }

        stage('Run Automated Tests') {
            steps {
                script {
                    // Ждем, пока сервисы развернутся и будут готовы
                    sh "sleep 60"

                    // Проверка доступности веб-сервера (фронтенда)
                    echo "Checking web server availability..."
                    sh "curl -f http://localhost:8080 || { echo 'Web server is not accessible!'; exit 1; }"
                    echo "Web server is accessible!"

                    // Проверка подключения к базе данных и наличия таблицы
                    echo "Checking database connectivity and table presence..."
                    def dbContainerId = sh(returnStdout: true, script: 'docker ps -q --filter ancestor=mhjkeee/mysql --filter status=running').trim()
                    
                    if (dbContainerId) {
                        // Используем docker exec для выполнения команды внутри контейнера базы данных
                        // Проверяем наличие таблицы 'products' в базе данных 'lena'
                        sh """docker exec ${dbContainerId} mysql -u root -p'secret' -e 'USE lena; DESCRIBE products;'"""
                    } else {
                        echo "Database container not found!"
                        error "Database connectivity test failed."
                    }
                    echo "Database connectivity and table 'products' test passed!"
                }
            }
        }
    }

    post {
        always {
            // Действия, которые выполняются всегда после завершения пайплайна
            cleanWs() // Очистка рабочего пространства
        }
        success {
            echo 'Pipeline finished successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}