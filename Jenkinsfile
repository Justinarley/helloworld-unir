pipeline {
    agent any

    environment {
        PYTHONPATH = "${WORKSPACE}"
        WIRE_JAR = "/var/lib/jenkins/tools/wiremock-standalone-3.13.2.jar"
        FLASK_APP = "app.api:api_application"
        JMETER_BIN = "/var/lib/jenkins/tools/apache-jmeter-5.6.3/bin/jmeter"
    }

    stages {
        stage('Setup') {
            steps {
                sh "fuser -k 5000/tcp || true"
                sh "fuser -k 9090/tcp || true"
                
                git branch: 'develop', url: 'https://github.com/Justinarley/helloworld-unir.git'
                sh 'python3 -m venv venv_jenkins'
                sh './venv_jenkins/bin/pip install -r requirements.txt'
            }
        }

        stage('Quality & Coverage') {
            steps {
                sh './venv_jenkins/bin/coverage run -m pytest test/unit --junitxml=results_unit.xml'
                sh './venv_jenkins/bin/coverage xml -o coverage.xml'
            }
            post {
                always {
                    junit 'results_unit.xml'
                    recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']],
                        qualityGates: [
                            [threshold: 95.0, metric: 'LINE', unstable: true],
                            [threshold: 85.0, metric: 'LINE', critical: true]
                        ])
                }
            }
        }

        stage('Static Analysis') {
            parallel {
                stage('Flake8') {
                    steps {
                        sh './venv_jenkins/bin/flake8 app/ --output-file=flake8.log || true'
                        recordIssues tools: [pyLint(pattern: 'flake8.log')],
                                     qualityGates: [[threshold: 8, type: 'TOTAL', qualityGateType: 'UNSTABLE']]
                    }
                }
                stage('Bandit') {
                    steps {
                        sh './venv_jenkins/bin/bandit -r app/ -f json -o bandit.json || true'
                        recordIssues tools: [issues(pattern: 'bandit.json', id: 'bandit', name: 'Bandit')],
                                     qualityGates: [[threshold: 2, type: 'TOTAL', qualityGateType: 'UNSTABLE']]
                    }
                }
            }
        }

        stage('Integration & Performance') {
            steps {
                script {
                    sh "java -jar ${WIRE_JAR} --root-dir ${WORKSPACE}/test/wiremock --port 9090 &"
                    sh "PYTHONPATH=. ./venv_jenkins/bin/flask run --host=0.0.0.0 --port=5000 &"
                    sh "sleep 10"
                    
                    try {
                        sh "PYTHONPATH=. ./venv_jenkins/bin/pytest test/rest"
                        sh "${JMETER_BIN} -n -t test/jmeter/flask.jmx -l resultados_performance.jtl"
                    } finally {
                        sh "fuser -k 5000/tcp || true"
                        sh "fuser -k 9090/tcp || true"
                        perfReport sourceDataFiles: 'resultados_performance.jtl'
                    }
                }
            }
        }
    }
}