pipeline {
    agent none

    environment {
        IMAGEN = "marinagr17/myapp"
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
                    args '-u root:root'
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

                    # Usar SQLite en CI
                    export DB_ENGINE=django.db.backends.sqlite3
                    export DB_NAME=":memory:"
                    export DB_HOST=""

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
                            sh 'python3 manage.py --version || true'
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
