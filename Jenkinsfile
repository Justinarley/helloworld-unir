pipeline {
    agent any

    environment {
        PYTHONPATH = "${WORKSPACE}"
        WIRE_JAR = "/var/lib/jenkins/tools/wiremock-standalone-3.13.2.jar"
        FLASK_APP = "app.api:api_application"
    }

    stages {
        stage('Get Code') {
            steps {
                // Obtenemos el código de la rama develop
                git branch: 'develop', url: 'https://github.com/Justinarley/helloworld-unir.git'
                sh 'python3 -m venv venv_jenkins'
                sh './venv_jenkins/bin/pip install -r requirements.txt flake8 bandit coverage'
            }
        }

        stage('Unit & Coverage') {
            steps {
                // Ejecutamos las pruebas unitarias y generamos los datos de cobertura
                // La guía prohíbe ejecutar las pruebas unitarias dos veces, así que lo hacemos aquí
                sh './venv_jenkins/bin/coverage run -m pytest test/unit --junitxml=results_unit.xml'
                sh './venv_jenkins/bin/coverage xml -o coverage.xml'
            }
            post {
                always {
                    junit 'results_unit.xml'
                    // Configuramos los baremos del Reto 1:
                    // Líneas: 85-95 Unstable. Ramas: 80-90 Unstable.
                    recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']],
                        qualityGates: [
                            [threshold: 95.0, metric: 'LINE', baseline: 'PROJECT', unstable: true],
                            [threshold: 85.0, metric: 'LINE', baseline: 'PROJECT', critical: true],
                            [threshold: 90.0, metric: 'BRANCH', baseline: 'PROJECT', unstable: true],
                            [threshold: 80.0, metric: 'BRANCH', baseline: 'PROJECT', critical: true]
                        ])
                }
            }
        }

        stage('Static (Flake8)') {
            steps {
                sh './venv_jenkins/bin/flake8 app/ --output-file=flake8.log || true'
            }
            post {
                always {
                    // Baremo: 8 hallazgos Unstable, 10 hallazgos Failure
                    recordIssues tool: flake8(pattern: 'flake8.log'), 
                                 qualityGates: [[threshold: 8, type: 'TOTAL', qualityGateType: 'UNSTABLE'],
                                               [threshold: 10, type: 'TOTAL', qualityGateType: 'FAILURE']]
                }
            }
        }

        stage('Security Test (Bandit)') {
            steps {
                // IMPORTANTE: Si 'bandit' sigue fallando como método DSL, usa:
                // recordIssues tool: issues(pattern: 'bandit.log', id: 'bandit', name: 'Bandit')
                sh './venv_jenkins/bin/bandit -r app/ -f txt -o bandit.log || true'
            }
            post {
                always {
                    // Baremo: 2 hallazgos Unstable, 4 hallazgos Failure
                    recordIssues tool: issues(pattern: 'bandit.log', id: 'bandit', name: 'Bandit'), 
                                 qualityGates: [[threshold: 2, type: 'TOTAL', qualityGateType: 'UNSTABLE'],
                                               [threshold: 4, type: 'TOTAL', qualityGateType: 'FAILURE']]
                }
            }
        }

        stage('Rest') {
            steps {
                script {
                    // Levantamos servicios para pruebas de integración
                    sh "java -jar ${WIRE_JAR} --root-dir ${WORKSPACE}/test/wiremock --port 9090 &"
                    sh "PYTHONPATH=. ./venv_jenkins/bin/flask run --host=0.0.0.0 --port=5000 &"
                    sh "sleep 10"
                    try {
                        sh "PYTHONPATH=. ./venv_jenkins/bin/pytest test/rest"
                    } finally {
                        sh "fuser -k 5000/tcp || true"
                        sh "fuser -k 9090/tcp || true"
                    }
                }
            }
        }

        stage('Performance') {
            steps {
                script {
                    // Solo Flask para JMeter (Wiremock no es necesario según la guía)
                    sh "PYTHONPATH=. ./venv_jenkins/bin/flask run --host=0.0.0.0 --port=5000 &"
                    sh "sleep 5"
                    try {
                        sh "jmeter -n -t test/jmeter/flask.jmx -l resultados_performance.jtl"
                    } finally {
                        sh "fuser -k 5000/tcp || true"
                        perfReport sourceDataFiles: 'resultados_performance.jtl'
                    }
                }
            }
        }
    }
}