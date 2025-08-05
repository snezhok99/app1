pipeline {
    agent any

    environment {
        STACK_NAME = 'app'
        DB_SERVICE = 'db'
        DB_USER = 'root'
        DB_PASSWORD = 'secret'
        DB_NAME = 'lena'
        FRONTEND_URL = 'http://localhost:8080' // или IP manager-ноды, если запускаешь Jenkins вне неё
    }

    stages {
        stage('Wait for Services') {
            steps {
                echo 'Ожидание поднятия сервисов...'
                sleep time: 20, unit: 'SECONDS'
            }
        }

        stage('Test DB Connection') {
            steps {
                echo 'Проверка подключения к базе данных...'
                script {
                    def dbContainerId = sh(
                        script: "docker ps --filter 'name=${STACK_NAME}_${DB_SERVICE}' --format '{{.ID}}'",
                        returnStdout: true
                    ).trim()

                    if (!dbContainerId) {
                        error("Контейнер базы данных не найден!")
                    }

                    sh """
                        docker exec ${dbContainerId} \
                        mysql -u${DB_USER} -p${DB_PASSWORD} -e 'SELECT 1;'
                    """
                }
            }
        }

        stage('Test DB Tables') {
            steps {
                echo 'Проверка наличия таблиц в базе данных...'
                script {
                    def dbContainerId = sh(
                        script: "docker ps --filter 'name=${STACK_NAME}_${DB_SERVICE}' --format '{{.ID}}'",
                        returnStdout: true
                    ).trim()

                    sh """
                        docker exec ${dbContainerId} \
                        mysql -u${DB_USER} -p${DB_PASSWORD} -e "USE ${DB_NAME}; SHOW TABLES;"
                    """
                }
            }
        }

        stage('Test Frontend to Backend') {
            steps {
                echo 'Проверка связи между фронтом и бэком...'
                sh """
                    curl -fsS ${FRONTEND_URL} || {
                        echo 'Ошибка: фронтенд недоступен или не может связаться с бэком.'
                        exit 1
                    }
                """
            }
        }
    }

    post {
        success {
            echo '✅ Все тесты пройдены успешно!'
        }
        failure {
            echo '❌ Пайплайн завершился с ошибкой!'
        }
    }
}
