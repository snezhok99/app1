pipeline {
    agent any

    environment {
        DOCKER_USERNAME = 'snezhok99'
        DOCKER_CREDENTIALS_ID = 'docker-hub-creds'
        SWARM_STACK_NAME = 'app'
        DB_SERVICE = 'app_db'
        DB_USER = 'root'
        DB_PASSWORD = 'secret'
        DB_NAME = 'lena'
        FRONTEND_URL = 'http://localhost:8080' // укажи IP, если Jenkins на другой машине
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKER_CREDENTIALS_ID,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    script {
                        sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'

                        // Сборка
                        sh "docker build -f php.Dockerfile -t ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-web:latest ."
                        sh "docker build -f mysql.Dockerfile -t ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-db:latest ."

                        // Публикация
                        sh "docker push ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-web:latest"
                        sh "docker push ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-db:latest"
                    }
                }
            }
        }

        stage('Deploy to Docker Swarm') {
            steps {
                script {
                    sh "docker stack deploy --with-registry-auth -c docker-compose.yaml ${SWARM_STACK_NAME}"
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    echo '⏳ Ожидаем запуск сервисов...'
                    sleep time: 30, unit: 'SECONDS'

                    echo '🧪 Проверка фронтенда...'
                    sh """
                        curl -fsS ${FRONTEND_URL}  {
                            echo '❌ Фронт не отвечает!'
                            exit 1
                        }
                    """

                    echo '🧪 Проверка БД...'
                    def dbContainerId = sh(
                        script: "docker ps --filter 'name=${SWARM_STACK_NAME}_${DB_SERVICE}' --format '{{.ID}}'",
                        returnStdout: true
                    ).trim()

                    if (!dbContainerId) {
                        error("❌ Контейнер базы данных не найден!")
                    }

                    echo '🧪 Подключение к БД...'
                    sh """
                        docker exec ${dbContainerId} \
                        mysql -u${DB_USER} -p${DB_PASSWORD} -e 'SELECT 1;'  exit 1
                    """

                    echo '🧪 Проверка таблиц...'
                    sh """
                        docker exec ${dbContainerId} \
                        mysql -u${DB_USER} -p${DB_PASSWORD} -e 'USE ${DB_NAME}; SHOW TABLES;' || exit 1
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ Все этапы выполнены успешно!'
        }
        failure {
            echo '❌ Сборка или тесты не прошли. Проверь логи выше.'
        }
        always {
            cleanWs()
        }
    }
}
