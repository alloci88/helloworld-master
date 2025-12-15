pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        VENV_DIR = ".venv"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                bat '''
                    whoami
                    hostname
                    dir
                '''
            }
        }

        stage('Prepare env') {
            steps {
                bat '''
                    @echo on
                    py -3 --version
                    py -3 -m venv %VENV_DIR%
                    call %VENV_DIR%\\Scripts\\activate.bat
                    python -m pip install --upgrade pip
                    if exist requirements.txt pip install -r requirements.txt
                    if exist requirements-dev.txt pip install -r requirements-dev.txt
                    pip install pytest pytest-cov junit-xml requests
                '''
            }
        }

        stage('Build (simulated)') {
            steps {
                bat '''
                    @echo on
                    echo Simulando build (Python interpretado)
                '''
            }
        }

        stage('Unit tests') {
            steps {
                bat '''
                    @echo on
                    call %VENV_DIR%\\Scripts\\activate.bat
                    if not exist reports mkdir reports
                    pytest -q test\\unit --junitxml=reports\\junit-unit.xml
                '''
            }
            post {
                always {
                    junit testResults: 'reports/junit-unit.xml', allowEmptyResults: true
                }
            }
        }

        stage('Integration tests (REST)') {
            steps {
                bat '''
                    @echo on
                    call %VENV_DIR%\\Scripts\\activate.bat
                    if not exist reports mkdir reports
                    pytest -q test\\rest --junitxml=reports\\junit-rest.xml
                '''
            }
            post {
                always {
                    junit testResults: 'reports/junit-rest.xml', allowEmptyResults: true
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/*.xml', fingerprint: true, allowEmptyArchive: true
        }
    }
}
