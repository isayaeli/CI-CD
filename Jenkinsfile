pipeline {
    agent any
    options {
        disableConcurrentBuilds()
    }
    environment {
        DJANGO_SETTINGS_MODULE = 'store.settings'
        PYTHONUNBUFFERED = '1'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build and Test') {
            failFast false
            parallel {
                stage('Python 3.10') {
                    agent {
                        docker {
                            image 'python:3.10-slim'
                            reuseNode true
                        }
                    }
                    steps {
                        sh 'python -m pip install --upgrade pip'
                        sh 'pip install -r requirements.txt'
                        sh 'python manage.py test'
                    }
                }
                stage('Python 3.11') {
                    agent {
                        docker {
                            image 'python:3.11-slim'
                            reuseNode true
                        }
                    }
                    steps {
                        sh 'python -m pip install --upgrade pip'
                        sh 'pip install -r requirements.txt'
                        sh 'python manage.py test'
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}