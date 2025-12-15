pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        VENV_DIR = ".venv"
        FLASK_PORT = "5000"
        WIREMOCK_PORT = "9090"
        WIREMOCK_DIR = "test\\wiremock"
        WIREMOCK_JAR = "test\\wiremock\\wiremock-standalone.jar"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                bat '''
                    @echo on
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

                    pip install pytest pytest-cov junit-xml requests flask

                    rem Asegura imports del proyecto
                    set PYTHONPATH=%CD%
                    python -c "import app; print('app import OK')"
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
                    set PYTHONPATH=%CD%

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

        stage('Start services') {
            steps {
                bat '''
                    @echo on
                    call %VENV_DIR%\\Scripts\\activate.bat
                    set PYTHONPATH=%CD%

                    rem --- Flask API ---
                    start "flask-api" /B cmd /c "set FLASK_APP=app\\api.py && flask run --host=127.0.0.1 --port=%FLASK_PORT% > flask.log 2>&1"

                    rem --- WireMock ---
                    if exist "%WIREMOCK_JAR%" (
                        start "wiremock" /B cmd /c "java -jar %WIREMOCK_JAR% --port %WIREMOCK_PORT% --root-dir %WIREMOCK_DIR% > wiremock.log 2>&1"
                    ) else (
                        echo Wiremock jar not found at %WIREMOCK_JAR%
                        exit /b 2
                    )

                    rem --- Wait until ports are ready ---
                    python - <<PY
import socket, time, sys

def wait_port(port, host="127.0.0.1", timeout_s=30):
    start = time.time()
    while time.time() - start < timeout_s:
        try:
            with socket.create_connection((host, port), timeout=1):
                return True
        except OSError:
            time.sleep(0.5)
    return False

ok_flask = wait_port(int("%FLASK_PORT%"))
ok_wire  = wait_port(int("%WIREMOCK_PORT%"))

print("Flask ready:", ok_flask)
print("WireMock ready:", ok_wire)

if not (ok_flask and ok_wire):
    sys.exit(1)
PY
                '''
            }
        }

        stage('Integration tests (REST)') {
            steps {
                bat '''
                    @echo on
                    call %VENV_DIR%\\Scripts\\activate.bat
                    set PYTHONPATH=%CD%

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
            bat '''
                @echo on
                rem cleanup best-effort
                taskkill /F /IM java.exe /T >NUL 2>&1
                taskkill /F /IM python.exe /T >NUL 2>&1
            '''
            archiveArtifacts artifacts: 'reports/*.xml,*.log', fingerprint: true, allowEmptyArchive: true
        }
    }
}
