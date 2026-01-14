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
                // 1. Obtenemos el código
                git branch: 'develop', url: 'https://github.com/Justinarley/helloworld-unir.git'
                // 2. Preparamos el entorno e instalamos TODO lo del requirements.txt
                sh 'python3 -m venv venv_jenkins'
                sh './venv_jenkins/bin/pip install -r requirements.txt'
            }
        }

        stage('Unit') {
            steps {
                // Ejecutamos pruebas unitarias (se ejecutan una sola vez como pide la guía)
                sh './venv_jenkins/bin/coverage run -m pytest test/unit --junitxml=results_unit.xml'
                sh './venv_jenkins/bin/coverage xml -o coverage.xml'
            }
            post {
                always {
                    junit 'results_unit.xml'
                }
            }
        }

        stage('Coverage') {
            steps {
                // Aplicamos los baremos: Líneas (85-95) y Ramas (80-90)
                recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']],
                    qualityGates: [
                        [threshold: 95.0, metric: 'LINE', unstable: true],
                        [threshold: 85.0, metric: 'LINE', critical: true],
                        [threshold: 90.0, metric: 'BRANCH', unstable: true],
                        [threshold: 80.0, metric: 'BRANCH', critical: true]
                    ])
            }
        }

        stage('Static (Flake8)') {
            steps {
                sh './venv_jenkins/bin/flake8 app/ --output-file=flake8.log || true'
            }
            post {
                always {
                    // Usamos el ID genérico 'pylint' que todos los Jenkins tienen para Flake8
                    recordIssues tools: [pyLint(pattern: 'flake8.log')],
                                 qualityGates: [[threshold: 8, type: 'TOTAL', qualityGateType: 'UNSTABLE'],
                                               [threshold: 10, type: 'TOTAL', qualityGateType: 'FAILURE']]
                }
            }
        }

        stage('Security Test (Bandit)') {
            steps {
                sh './venv_jenkins/bin/bandit -r app/ -f json -o bandit.json || true'
            }
            post {
                always {
                    // Usamos 'issues' con el ID manual para que no busque el método bandit()
                    recordIssues tools: [issues(pattern: 'bandit.json', id: 'bandit', name: 'Bandit')],
                                 qualityGates: [[threshold: 2, type: 'TOTAL', qualityGateType: 'UNSTABLE'],
                                               [threshold: 4, type: 'TOTAL', qualityGateType: 'FAILURE']]
                }
            }
        }

        stage('Rest') {
            steps {
                script {
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