pipeline {
    agent none
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }
    
    environment {
        IMAGEN = "marinagr17/dockercd-cd"
        USUARIO = credentials('USER_DOCKERHUB')
        REPO = "https://github.com/marinagr17/Guestbook-Tutorial.git"
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
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], userRemoteConfigs: [[url: '${REPO}']]])
                
                dir("build/app") {
                    sh '''
                    apt-get update && apt-get install -y --no-install-recommends gcc default-libmysqlclient-dev build-essential libssl-dev pkg-config
                    pip install --quiet -r requirements.txt
                    python3 manage.py test --settings=django_tutorial.settings_test --verbosity=2
                    '''
                }
            }
        }
        
        stage("Build & Push") {
            agent any
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], userRemoteConfigs: [[url: '${REPO}']]])
                
                script {
                    dir('build') {
                        docker.build("${IMAGEN}:${BUILD_NUMBER}")
                        docker.image("${IMAGEN}:${BUILD_NUMBER}").inside('-u root') {
                            sh 'python3 -c "import django; print(django.get_version())"'
                        }
                    }
                    
                    sh '''
                    echo ${USUARIO_PSW} | docker login -u ${USUARIO_USR} --password-stdin
                    docker push ${IMAGEN}:${BUILD_NUMBER}
                    docker tag ${IMAGEN}:${BUILD_NUMBER} ${IMAGEN}:latest
                    docker push ${IMAGEN}:latest
                    docker logout
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh '''
            docker rmi ${IMAGEN}:${BUILD_NUMBER} 2>/dev/null || true
            docker rmi ${IMAGEN}:latest 2>/dev/null || true
            '''
            cleanWs()
        }
    }
}