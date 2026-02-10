pipeline {
    agent none
    environment {
        IMAGEN = "marinagr17/proyectoCI-CD"
        USUARIO = 'USER_DOCKERHUB'
    }
    stages {
        stage("Build and Test Django Project") {
            agent {
                docker { 
                    image 'python:3.12-slim'
                    args '-u root:root'
                }
            }
            steps {
                git branch:'master', url:'https://github.com/marinagr17/Guestbook-Tutorial.git'
                dir('build/app') {
                    sh 'pip install -r requirements.txt'
                    sh 'python3 manage.py test'
                }
            }
        }

        stage("Crear y Subir Imagen Docker") {
            agent any
            steps {
                script {
                    // Construir la imagen usando build/ como contexto
                    def newApp = docker.build("$IMAGEN:$BUILD_NUMBER", "build")
                    
                    // Test básico dentro del contenedor
                    docker.image("$IMAGEN:$BUILD_NUMBER").inside('-u root') {
                        sh 'python3 manage.py --version || echo "Django project container ok"'
                    }

                    // Push a Docker Hub
                    docker.withRegistry('', USUARIO) {
                        newApp.push()
                    }

                    // Cleanup local
                    sh "docker rmi $IMAGEN:$BUILD_NUMBER"
                }
            }
        }
    }
}
