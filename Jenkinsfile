pipeline {
    agent any
    stages {
        stage('Test Docker') {
            steps {
                sh '''
                    docker --version
                    docker ps
                '''
            }
        }
    }
}