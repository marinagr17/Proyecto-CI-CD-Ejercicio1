pipeline {
    agent none
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }
    
    environment {
        IMAGEN = "marinagr17/dockerci-cd"
        DOCKERFILE_PATH = "build"
        APP_PATH = "build/app"
    }
    
    stages {
        stage("Checkout & Test Django") {
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
                        userRemoteConfigs: [[url: 'https://github.com/marinagr17/Proyecto-CI-CD-Ejercicio1.git']]])
                    
                    // Verificar que todo está
                    sh '''
                    echo "✓ Repositorio clonado"
                    echo ""
                    echo "Estructura:"
                    ls -la
                    '''
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
                        
                        echo "Ejecutando tests de Django..."
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
                    def imageTag = "${IMAGEN}:${BUILD_NUMBER}".toLowerCase()
                    
                    echo "Construyendo imagen: ${imageTag}"
                    
                    // Verificar que Dockerfile existe en el workspace
                    sh '''
                    echo "Verificando archivos..."
                    test -f ${DOCKERFILE_PATH}/Dockerfile && echo "✓ Dockerfile encontrado" || exit 1
                    test -d ${APP_PATH} && echo "✓ app/ encontrado" || exit 1
                    ls -lh ${DOCKERFILE_PATH}/
                    '''
                    
                    // Construir imagen - el contexto es "build"
                    def newApp = docker.build(imageTag, "${DOCKERFILE_PATH}")
                    
                    echo "Validando Django en contenedor..."
                    docker.image(imageTag).inside('-u root') {
                        dir('app') {
                            sh '''
                            python3 -c "import django; print(f'✓ Django {django.get_version()}')"
                            '''
                        }
                    }
                    
                    env.BUILT_IMAGE = imageTag
                    echo "✓ Imagen construida: ${imageTag}"
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
                    
                    withCredentials([usernamePassword(
                        credentialsId: 'USER_DOCKERHUB',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                        # Login seguro
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        
                        # Push con BUILD_NUMBER
                        docker push "${BUILT_IMAGE}"
                        
                        # Push con latest
                        LATEST_TAG=$(echo "${BUILT_IMAGE}" | sed 's/:.*/:latest/')
                        docker tag "${BUILT_IMAGE}" "${LATEST_TAG}"
                        docker push "${LATEST_TAG}"
                        
                        # Logout
                        docker logout
                        
                        echo "✓ Imágenes publicadas:"
                        echo "  - ${BUILT_IMAGE}"
                        echo "  - ${LATEST_TAG}"
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                sh '''
                docker rmi "${BUILT_IMAGE}" 2>/dev/null || true
                docker rmi "$(echo "${BUILT_IMAGE}" | sed 's/:.*/:latest/')" 2>/dev/null || true
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
