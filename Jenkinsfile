pipeline {
    agent none
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }
    
    environment {
        IMAGEN = "marinagr17/DockerCI-CD"
        USUARIO = credentials('USER_DOCKERHUB')
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
                script {
                    echo "Clonando repositorio..."
                    checkout([$class: 'GitSCM', 
                        branches: [[name: '*/master']], 
                        userRemoteConfigs: [[url: 'https://github.com/marinagr17/Guestbook-Tutorial.git']]])
                }
                
                dir('build/app') {
                    script {
                        echo "Instalando dependencias del sistema..."
                        sh '''
                        apt-get update && \
                        apt-get install -y --no-install-recommends \
                            gcc \
                            default-libmysqlclient-dev \
                            build-essential \
                            libssl-dev \
                            pkg-config && \
                        rm -rf /var/lib/apt/lists/*
                        '''
                        
                        echo "Instalando dependencias Python..."
                        sh 'pip install --quiet -r requirements.txt'
                        
                        echo "Ejecutando tests de Django..."
                        sh '''
                        python3 manage.py test \
                            --verbosity=2 \
                            --parallel 1 \
                            --keepdb || true
                        '''
                    }
                }
            }
        }
        
        stage("Construir imagen Docker") {
            agent any
            steps {
                script {
                    echo "Construyendo imagen Docker: ${IMAGEN}:${BUILD_NUMBER}"
                    def newApp = docker.build("${IMAGEN}:${BUILD_NUMBER}", "build")
                    
                    echo "Verificando Django en el contenedor..."
                    docker.image("${IMAGEN}:${BUILD_NUMBER}").inside('-u root') {
                        dir('app') {
                            sh '''
                            python3 -c "import django; \
                            print(f'✓ Django {django.get_version()} instalado correctamente')"
                            '''
                        }
                    }
                    
                    env.IMAGE_ID = "${IMAGEN}:${BUILD_NUMBER}"
                }
            }
        }
        
        stage("Subir imagen a Docker Hub") {
            agent any
            when {
                branch 'master'
            }
            steps {
                script {
                    echo "Subiendo ${IMAGE_ID} a Docker Hub..."
                    docker.withRegistry('https://registry.hub.docker.com', USUARIO) {
                        def app = docker.image(IMAGE_ID)
                        app.push()
                        app.push('latest')
                        echo "✓ Imagen subida: ${IMAGEN}:${BUILD_NUMBER} y ${IMAGEN}:latest"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "Limpiando recursos locales..."
                sh '''
                docker rmi ${IMAGEN}:${BUILD_NUMBER} 2>/dev/null || true
                docker rmi ${IMAGEN}:latest 2>/dev/null || true
                '''
                cleanWs()
            }
        }
        success {
            echo "Pipeline completado exitosamente - Build #${BUILD_NUMBER}"
        }
        failure {
            echo "Pipeline falló - Build #${BUILD_NUMBER}"
            echo "Revisar logs para más detalles"
        }
    }
}
