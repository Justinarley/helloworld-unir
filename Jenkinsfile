pipeline {
    agent any

    environment {
        PYTHONPATH = "${WORKSPACE}"
        WIRE_JAR = "/var/lib/jenkins/tools/wiremock-standalone-3.13.2.jar"
        FLASK_APP = "app.api:api_application"
    }

    stages {
        stage('Checkout y Preparación') {
            steps {
                git branch: 'develop', url: 'https://github.com/Justinarley/helloworld-unir.git'
                sh 'python3 -m venv venv_jenkins'
                sh './venv_jenkins/bin/pip install -r requirements.txt flake8 bandit coverage'
            }
        }

        stage('Tests Unitarios y Cobertura') {
            steps {
                sh './venv_jenkins/bin/coverage run -m pytest test/unit --junitxml=results_unit.xml'
                sh './venv_jenkins/bin/coverage xml -o coverage.xml'
            }
            post {
                always {
                    junit 'results_unit.xml'
                    recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']])
                }
            }
        }

        stage('Static Analysis (Flake8)') {
            steps {
                sh './venv_jenkins/bin/flake8 app/ --output-file=flake8.log || true'
            }
            post {
                always {
                    // Usamos la sintaxis de 'tool' que es más robusta
                    recordIssues tool: flake8(pattern: 'flake8.log'), 
                                 qualityGates: [[threshold: 8, type: 'TOTAL', qualityGateType: 'UNSTABLE'],
                                               [threshold: 10, type: 'TOTAL', qualityGateType: 'FAILURE']]
                }
            }
        }

        stage('Security (Bandit)') {
            steps {
                sh './venv_jenkins/bin/bandit -r app/ -f txt -o bandit.txt || true'
            }
            post {
                always {
                    // Cambiamos a 'pyLint' con el patrón de bandit o usamos el ID genérico
                    // Esta sintaxis suele evitar el error "No such DSL method bandit"
                    recordIssues tool: bandit(pattern: 'bandit.txt'), 
                                 qualityGates: [[threshold: 2, type: 'TOTAL', qualityGateType: 'UNSTABLE'],
                                               [threshold: 4, type: 'TOTAL', qualityGateType: 'FAILURE']]
                }
            }
        }

        stage('Tests REST y Performance') {
            steps {
                script {
                    sh "java -jar ${WIRE_JAR} --root-dir ${WORKSPACE}/test/wiremock --port 9090 &"
                    sh "PYTHONPATH=. ./venv_jenkins/bin/flask run --host=0.0.0.0 --port=5000 &"
                    
                    echo 'Esperando a que los servidores levanten...'
                    sh "sleep 10"
                    
                    try {
                        sh "PYTHONPATH=. ./venv_jenkins/bin/pytest test/rest"
                        // Ejecución de JMeter
                        sh "jmeter -n -t test/jmeter/flask.jmx -l resultados_performance.jtl"
                    } finally {
                        sh "fuser -k 5000/tcp || true"
                        sh "fuser -k 9090/tcp || true"
                        // Publicación de reporte de rendimiento
                        perfReport sourceDataFiles: 'resultados_performance.jtl'
                    }
                }
            }
        }
    }
}