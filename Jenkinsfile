pipeline {
    agent none
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }
    
    environment {
        IMAGE_NAME = "marinagr17/dockerci-cd"
    }
    
    stages {
        stage("Checkout") {
            agent any
            steps {
                script {
                    echo "Clonando repositorio..."
                    checkout([$class: 'GitSCM', 
                        branches: [[name: '*/master']], 
                        userRemoteConfigs: [[url: 'https://github.com/marinagr17/Proyecto-CI-CD-Ejercicio1.git']]])
                    
                    // Mostrar estructura
                    sh '''
                    echo "✓ Repositorio clonado"
                    echo ""
                    echo "Estructura:"
                    ls -la
                    echo ""
                    echo "Contenido build/:"
                    ls -la build/
                    '''
                }
            }
        }
        
        stage("Test Django") {
            agent {
                docker {
                    image 'python:3.12-slim'
                    args '-u root:root'
                }
            }
            steps {
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
                    sh 'pip install --quiet -r build/app/requirements.txt'
                    
                    echo "Ejecutando tests de Django..."
                    sh '''
                    cd build/app && \
                    python3 manage.py test \
                        --settings=django_tutorial.settings_test \
                        --verbosity=2
                    '''
                }
            }
        }
        
        stage("Build Docker Image") {
            agent any
            steps {
                script {
                    def imageTag = "${IMAGE_NAME}:${BUILD_NUMBER}".toLowerCase()
                    
                    echo "Construyendo imagen: ${imageTag}"
                    
                    // Verificar que Dockerfile existe
                    sh '''
                    echo "Verificando archivos antes de build..."
                    test -f build/Dockerfile && echo "✓ build/Dockerfile existe" || exit 1
                    test -d build/app && echo "✓ build/app existe" || exit 1
                    test -f build/entrypoint.sh && echo "✓ build/entrypoint.sh existe" || exit 1
                    '''
                    
                    // Construir imagen Docker
                    // El contexto es "build" así Docker busca build/Dockerfile
                    def builtImage = docker.build(imageTag, "build")
                    
                    echo "Validando Django en la imagen..."
                    docker.image(imageTag).inside('-u root') {
                        dir('app') {
                            sh '''
                            python3 -c "import django; print(f'✓ Django {django.get_version()} OK')"
                            '''
                        }
                    }
                    
                    env.BUILT_IMAGE = imageTag
                    echo "✓ Imagen construida exitosamente: ${imageTag}"
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
                        
                        # Push imagen con BUILD_NUMBER
                        docker push "${BUILT_IMAGE}"
                        echo "✓ Publicada: ${BUILT_IMAGE}"
                        
                        # Crear tag latest
                        LATEST_TAG=$(echo "${BUILT_IMAGE}" | sed 's/:.*/:latest/')
                        docker tag "${BUILT_IMAGE}" "${LATEST_TAG}"
                        docker push "${LATEST_TAG}"
                        echo "✓ Publicada: ${LATEST_TAG}"
                        
                        # Logout
                        docker logout
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo Limpiando recursos..."
                sh '''
                docker rmi "${BUILT_IMAGE}" 2>/dev/null || true
                docker rmi "$(echo "${BUILT_IMAGE}" | sed 's/:.*/:latest/')" 2>/dev/null || true
                '''
                cleanWs()
            }
        }
        success {
            echo "Pipeline #${BUILD_NUMBER} completado exitosamente"
        }
        failure {
            echo "Pipeline #${BUILD_NUMBER} falló"
        }
    }
}
