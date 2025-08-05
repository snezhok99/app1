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
                    withCredentials([usernamePassword(
                        credentialsId: env.DOCKER_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    ]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin || exit 1
                        '''
                    }

                    sh "docker build -f php.Dockerfile -t ${env.DOCKER_USERNAME}/${env.SWARM_STACK_NAME}-web:latest ."
                    sh "docker build -f mysql.Dockerfile -t ${env.DOCKER_USERNAME}/${env.SWARM_STACK_NAME}-db:latest ."
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
                    sh "docker push ${env.DOCKER_USERNAME}/${env.SWARM_STACK_NAME}-web:latest"
                    sh "docker push ${env.DOCKER_USERNAME}/${env.SWARM_STACK_NAME}-db:latest"
                }
            }
        }

        stage('Deploy to Docker Swarm') {
            steps {
                script {
                    sh '''
                        if ! docker info | grep -q "Swarm: active"; then
                            docker swarm init --advertise-addr $(hostname -i)
                        fi
                    '''
                    sh "docker stack deploy --with-registry-auth -c docker-compose.yaml ${env.SWARM_STACK_NAME}"
                }
            }
        }

        stage('Run Automated Tests') {
            steps {
                script {
                    echo "Waiting for services to be ready..."
                    sleep(time: 60, unit: 'SECONDS')

                    echo "Testing web service..."
                    sh '''
                        if ! curl -fsS http://localhost:8080; then
                            echo "Web service test failed!"
                            exit 1
                        fi
                    '''

                    echo "Testing database connection..."
                    def dbContainer = sh(script: """
                        docker service ps ${env.SWARM_STACK_NAME}_db --format '{{.Name}}.{{.ID}}' | head -n 1
                    """, returnStdout: true).trim()
                    
                    sh """
                        docker exec ${dbContainer} mysql -u root -p'secret' -e 'USE lena; SHOW TABLES;'
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            echo 'Pipeline failed!'
        }
        success {
            echo 'Pipeline succeeded!'
        }
    }
}