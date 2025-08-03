pipeline {
    agent any

    environment {
        DOCKER_USERNAME = 'snezhok99'
        DOCKER_CREDENTIALS_ID = 'c6b31924-c8a4-4ab3-bfb6-6139260a7909'
        SWARM_STACK_NAME = 'app'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    // Вход в Docker Hub через Jenkins Credentials (безопасно)
                    withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID,
                                                     usernameVariable: 'DOCKER_USER',
                                                     passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        '''
                    }

                    // Сборка образов
                    sh "docker build -f php.Dockerfile . -t ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-web:latest"
                    sh "docker build -f mysql.Dockerfile . -t ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-db:latest"
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
                    sh "docker push ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-web:latest"
                    sh "docker push ${DOCKER_USERNAME}/${SWARM_STACK_NAME}-db:latest"
                }
            }
        }

        stage('Deploy to Docker Swarm') {
            steps {
                script {
                    // Инициализация Swarm, если не инициализирован
                    sh '''
                        if ! docker info | grep -q "Swarm: active"; then
                            docker swarm init
                        fi
                    '''

                    sh "docker stack deploy --with-registry-auth -c docker-compose.yaml ${SWARM_STACK_NAME}"
                }
            }
        }

        stage('Run Automated Tests') {
            steps {
                script {
                    echo "Waiting for services to be ready..."
                    sh "sleep 60"

                    echo "Checking web server availability..."
                    sh '''
                        if ! curl -f http://localhost:8080; then
                            echo "Web server is not accessible!"
                            exit 1
                        fi
                    '''
                    echo "Web server is accessible!"

                    echo "Checking database connectivity and table presence..."
                    def dbContainerId = sh(returnStdout: true, script: '''
                        docker ps -q --filter "ancestor=${DOCKER_USERNAME}/${SWARM_STACK_NAME}-db" --filter "status=running"
                    ''').trim()

                    if (dbContainerId) {
                        sh """
                            docker exec ${dbContainerId} mysql -u root -p'secret' -e 'USE lena; DESCRIBE products;'
                        """
                    } else {
                        error "Database container not found! Connectivity test failed."
                    }
                    echo "Database connectivity and table test passed!"
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline finished successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}