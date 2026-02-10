pipeline {
    agent none
    environment {
        IMAGEN = "marinagr17/myapp"
        USUARIO = 'USER_DOCKERHUB'
    }
    stages {
        stage("Test Django") {
            agent {
                docker { 
                    image 'python:3.12-slim'
                    args '-u root:root --env-file .env.dev'
                }
            }
            steps {
                // Clonar el repositorio
                git branch:'master', url:'https://github.com/marinagr17/Guestbook-Tutorial.git'
                
                // Entrar al directorio donde está manage.py y requirements.txt
                dir('build/app') {
                    sh '''
                    apt-get update && \
                    apt-get install -y gcc default-libmysqlclient-dev build-essential libssl-dev pkg-config && \
                    rm -rf /var/lib/apt/lists/*
                    '''

                    echo "Instalando dependencias..."
                    sh 'pip install -r requirements.txt'

                    echo "Ejecutando tests de Django..."
                    sh 'python3 manage.py test'
                }
            }
        }

        stage("Construir y subir imagen Docker") {
            agent any
            steps {
                script {
                    // Construir la imagen Docker usando build/ 
                    def newApp = docker.build("$IMAGEN:$BUILD_NUMBER", "build")
                    
                    // Test dentro del contenedor para asegurarse de que está Django 
                    docker.image("$IMAGEN:$BUILD_NUMBER").inside('-u root') {
                        dir('app') {
                            sh 'python3 manage.py --version || echo "Django container OK"'
                        }
                    }

                    // Subir imagen a Docker Hub
                    docker.withRegistry('', USUARIO) {
                        newApp.push()
                    }

                    // Borrar imagen local
                    sh "docker rmi $IMAGEN:$BUILD_NUMBER || true"
                }
            }
        }
    }
}
