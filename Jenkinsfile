pipeline {
    agent any

    environment {
        SWARM_STACK_NAME = 'app'
        DB_SERVICE = 'app_db'
        DB_USER = 'root'
        DB_PASSWORD = 'secret'
        DB_NAME = 'lena'
        FRONTEND_URL = 'http://192.168.0.1:8080'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    // Сборка локальных Docker-образов
                    sh "docker build -f php.Dockerfile -t app-web:latest ."
                    sh "docker build -f mysql.Dockerfile -t app-db:latest ."
                }
            }
        }

        stage('Deploy to Docker Swarm') {
            steps {
                script {
                    // Убедимся, что swarm активен
                    sh '''
                        if ! docker info | grep -q "Swarm: active"; then
                            docker swarm init  true
                        fi
                    '''

                    // Деплой через docker stack
                    sh "docker stack deploy --with-registry-auth -c docker-compose.yaml ${SWARM_STACK_NAME}"
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    echo '⏳ Ожидание запуска сервисов...'
                    sleep time: 30, unit: 'SECONDS'

                    echo '🧪 Проверка доступности фронта...'
                    sh """
                        if ! curl -fsS ${FRONTEND_URL}; then
                            echo '❌ Фронт недоступен!'
                            exit 1
                        fi
                    """

                    echo '🧪 Проверка подключения к БД...'
                    def dbContainer = sh(
                        script: "docker ps --filter name=${SWARM_STACK_NAME}_${DB_SERVICE} --format '{{.ID}}'",
                        returnStdout: true
                    ).trim()

                    if (!dbContainer) {
                        error("❌ Контейнер базы данных не найден!")
                    }

                    echo '🧪 Подключение к MySQL...'
                    sh """
                        docker exec ${dbContainer} mysql -u${DB_USER} -p${DB_PASSWORD} -e 'SELECT 1;'  exit 1
                    """

                    echo '🧪 Проверка наличия таблиц в базе данных...'
                    sh """
                        docker exec ${dbContainer} \
                        mysql -u${DB_USER} -p${DB_PASSWORD} -e 'USE ${DB_NAME}; SHOW TABLES;' || exit 1
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ Все этапы успешно завершены!'
        }
        failure {
            echo '❌ Ошибка в одном из этапов. Проверь логи выше.'
        }
        always {
            cleanWs()
        }
    }
}
