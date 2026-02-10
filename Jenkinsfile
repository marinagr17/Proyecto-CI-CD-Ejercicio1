pipeline {
    agent none
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }
    
    environment {
        IMAGEN = "marinagr17/dockercd-cd"
        USUARIO = credentials('USER_DOCKERHUB')
        DOCKERFILE_PATH = "build"
        APP_PATH = "build/app"
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
                
                dir("${APP_PATH}") {
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
                        
                        echo "Ejecutando tests de Django con SQLite..."
                        sh '''
                        python3 manage.py test \
                            --settings=django_tutorial.settings_test \
                            --verbosity=2
                        '''
                    }
                }
            }
        }
        
        stage("Build Docker Image") {
            agent any
            steps {
                script {
                    echo "Construyendo imagen: ${IMAGEN}:${BUILD_NUMBER}"
                    def newApp = docker.build("${IMAGEN}:${BUILD_NUMBER}", "${DOCKERFILE_PATH}")
                    
                    echo "Validando Django en contenedor..."
                    docker.image("${IMAGEN}:${BUILD_NUMBER}").inside('-u root') {
                        dir('app') {
                            sh '''
                            python3 -c "import django; \
                            print(f'✓ Django {django.get_version()}')"
                            '''
                        }
                    }
                    
                    env.NEW_IMAGE = "${IMAGEN}:${BUILD_NUMBER}"
                }
            }
        }
        
        stage("Push to Docker Hub") {
            agent any
            when {
                branch 'master'
            }
            steps {
                script {
                    echo "Subiendo imagen a Docker Hub..."
                    docker.withRegistry('https://registry.hub.docker.com', USUARIO) {
                        def app = docker.image(env.NEW_IMAGE)
                        app.push()
                        app.push('latest')
                        echo "✓ Imagen: ${IMAGEN}:${BUILD_NUMBER}"
                        echo "✓ Imagen: ${IMAGEN}:latest"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                sh '''
                docker rmi ${IMAGEN}:${BUILD_NUMBER} 2>/dev/null || true
                docker rmi ${IMAGEN}:latest 2>/dev/null || true
                '''
                cleanWs()
            }
        }
        success {
            echo "Build #${BUILD_NUMBER} completado exitosamente"
        }
        failure {
            echo "Build #${BUILD_NUMBER} falló"
        }
    }
}
