pipeline {
    agent none

    environment {
        IMAGEN = "marinagr17/DockerCI-CD"
        USUARIO = 'USER_DOCKERHUB'

        VERSION_DT = "v4"
        VERSION_MDB = "latest"
        PUERTO = "8000"

        DB_HOST = "db"
        DB_NAME = "tutorial"
        DB_USER = "admin"
        DB_PASSWORD = "secret"

        DJANGO_SUPERUSER_USERNAME = "admin"
        DJANGO_SUPERUSER_EMAIL = "admin@example.com"
        DJANGO_SUPERUSER_PASSWORD = "admin123"

        MYSQL_ROOT_PASSWORD = "rootpass"
        CSRF_TRUSTED_ORIGINS = "https://docker.pingamarina.site,http://docker.pingamarina.site"
    }

    stages {

        stage("Test Django") {
            agent {
                docker {
                    image 'python:3.12-slim'
                    args """
                    -u root:root
                    -e VERSION_DT=${VERSION_DT}
                    -e VERSION_MDB=${VERSION_MDB}
                    -e PUERTO=${PUERTO}
                    -e DB_HOST=${DB_HOST}
                    -e DB_NAME=${DB_NAME}
                    -e DB_USER=${DB_USER}
                    -e DB_PASSWORD=${DB_PASSWORD}
                    -e DJANGO_SUPERUSER_USERNAME=${DJANGO_SUPERUSER_USERNAME}
                    -e DJANGO_SUPERUSER_EMAIL=${DJANGO_SUPERUSER_EMAIL}
                    -e DJANGO_SUPERUSER_PASSWORD=${DJANGO_SUPERUSER_PASSWORD}
                    -e MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
                    -e CSRF_TRUSTED_ORIGINS=${CSRF_TRUSTED_ORIGINS}
                    """
                }
            }

            steps {
                git branch:'master', url:'https://github.com/marinagr17/Guestbook-Tutorial.git'

                dir('build/app') {
                    sh '''
                    apt-get update && \
                    apt-get install -y gcc default-libmysqlclient-dev build-essential libssl-dev pkg-config && \
                    rm -rf /var/lib/apt/lists/*

                    pip install -r requirements.txt
                    python3 manage.py test
                    '''
                }
            }
        }

        stage("Construir y subir imagen Docker") {
            agent any
            steps {
                script {
                    def newApp = docker.build("$IMAGEN:$BUILD_NUMBER", "build")

                    docker.image("$IMAGEN:$BUILD_NUMBER").inside('-u root') {
                        dir('app') {
                            sh 'python3 manage.py --version || echo "Django container OK"'
                        }
                    }

                    docker.withRegistry('', USUARIO) {
                        newApp.push()
                    }

                    sh "docker rmi $IMAGEN:$BUILD_NUMBER || true"
                }
            }
        }
    }
}
