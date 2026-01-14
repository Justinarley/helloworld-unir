pipeline {
    agent any

    environment {
        // Usamos la ruta absoluta del workspace para evitar el error de "app not found"
        PYTHONPATH = "${WORKSPACE}"
        WIRE_JAR = "/var/lib/jenkins/tools/wiremock-standalone-3.13.2.jar"
        FLASK_APP = "app.api:api_application"
    }

    stages {
        stage('Checkout y Preparación') {
            steps {
                git branch: 'develop', url: 'https://github.com/Justinarley/helloworld-unir.git'
                sh 'python3 -m venv venv_jenkins'
                // Instalamos herramientas adicionales para el Reto 1
                sh './venv_jenkins/bin/pip install -r requirements.txt flake8 bandit coverage'
            }
        }

        stage('Tests Unitarios y Cobertura') {
            steps {
                // Ejecutamos tests y generamos reportes de JUnit y Cobertura (XML)
                sh './venv_jenkins/bin/coverage run -m pytest test/unit --junitxml=results_unit.xml'
                sh './venv_jenkins/bin/coverage xml -o coverage.xml'
            }
            post {
                always {
                    junit 'results_unit.xml'
                    // Plugin Cobertura: Umbrales UNIR (Lineas 85-95, Ramas 80-90)
                    publishCoverage(adapters: [coberturaAdapter(path: 'coverage.xml')],
                        globalThresholds: [
                            [thresholdTarget: 'Line', unstableThreshold: 85, unhealthyThreshold: 0],
                            [thresholdTarget: 'Branch', unstableThreshold: 80, unhealthyThreshold: 0]
                        ])
                }
            }
        }

        stage('Análisis Estático (Flake8)') {
            steps {
                // Umbrales UNIR: 8 unstable / 10 failed
                sh './venv_jenkins/bin/flake8 app/ --output-file=flake8.log || true'
            }
            post {
                always {
                    recordIssues(tools: [flake8(pattern: 'flake8.log')], 
                                 unstableTotalAll: 8, 
                                 failedTotalAll: 10)
                }
            }
        }

        stage('Seguridad (Bandit)') {
            steps {
                // Umbrales UNIR: 2 unstable / 4 failed
                sh './venv_jenkins/bin/bandit -r app/ -f txt -o bandit.txt || true'
            }
            post {
                always {
                    recordIssues(tools: [bandit(pattern: 'bandit.txt')], 
                                 unstableTotalAll: 2, 
                                 failedTotalAll: 4)
                }
            }
        }

        stage('Tests REST y Performance') {
            steps {
                script {
                    // Levantamos servicios
                    sh "java -jar ${WIRE_JAR} --root-dir ${WORKSPACE}/test/wiremock --port 9090 &"
                    sh "PYTHONPATH=. ./venv_jenkins/bin/flask run --host=0.0.0.0 --port=5000 &"
                    
                    echo 'Esperando a que los servidores levanten...'
                    sh "sleep 10"
                    
                    try {
                        // 1. Tests REST
                        sh "PYTHONPATH=. ./venv_jenkins/bin/pytest test/rest"
                        
                        // 2. Performance (JMeter) - Usando tu ruta de archivo
                        sh "jmeter -n -t test/jmeter/flask.jmx -l resultados_performance.jtl"
                        
                    } finally {
                        sh "fuser -k 5000/tcp || true"
                        sh "fuser -k 9090/tcp || true"
                        // Publicamos reporte de JMeter
                        perfReport sourceDataFiles: 'resultados_performance.jtl'
                    }
                }
            }
        }
    }
}