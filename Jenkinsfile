pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                // Получаем код из Git (по умолчанию текущий репозиторий)
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Собираем проект...'
                // Здесь может быть сборка Docker или компиляция, если нужно
                sh 'echo "Сборка завершена успешно!"'
            }
        }

        stage('Test') {
            steps {
                echo 'Выполняем тесты...'
                // Пример простого теста: файл существует
                sh 'test -f docker-compose.yaml'
            }
        }
    }

    post {
        success {
            echo 'Пайплайн завершился успешно!'
        }
        failure {
            echo 'Пайплайн завершился с ошибкой!'
        }
        always {
            cleanWs()
        }
    }
}