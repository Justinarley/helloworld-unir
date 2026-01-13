pipeline {
    agent any

    environment {
        PYTHONPATH = "."
        WIRE_JAR = "/var/lib/jenkins/tools/wiremock-standalone-3.13.2.jar"
        FLASK_APP = "app.api:api_application"
    }

    stages {
        stage('Checkout y Preparación') {
            steps {
                git branch: 'develop', url: 'https://github.com/Justinarley/helloworld-unir.git'
                sh 'python3 -m venv venv_jenkins'
                sh './venv_jenkins/bin/pip install -r requirements.txt'
            }
        }

        stage('Tests Unitarios') {
            steps {
                sh 'PYTHONPATH=. ./venv_jenkins/bin/pytest test/unit'
            }
        }

        stage('Tests REST (Integración)') {
            steps {
                script {
                    sh "java -jar ${WIRE_JAR} --root-dir ${WORKSPACE}/test/wiremock --port 9090 &"
                    
                    sh "PYTHONPATH=. ./venv_jenkins/bin/flask run --host=0.0.0.0 --port=5000 &"
                    
                    echo 'Esperando 10 segundos a que los servidores levanten...'
                    sh "sleep 10"
                    
                    try {
                        sh "PYTHONPATH=. ./venv_jenkins/bin/pytest test/rest"
                        echo '¡Tests REST superados con éxito!'
                    } finally {
                        echo 'Limpiando puertos 5000 y 9090...'
                        sh "fuser -k 5000/tcp || true"
                        sh "fuser -k 9090/tcp || true"
                    }
                }
            }
        }
    }
}