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
        WIREMOCK_JAR = "test\\wiremock\\mappings\\wiremock-standalone-3.13.2.jar"
        WAIT_TIMEOUT_SECONDS = "30"
        FLASK_PID_FILE = "flask.pid"
        WIREMOCK_PID_FILE = "wiremock.pid"
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

                    if exist %FLASK_PID_FILE% del /f /q %FLASK_PID_FILE%
                    if exist %WIREMOCK_PID_FILE% del /f /q %WIREMOCK_PID_FILE%

                    echo === Starting Flask on %FLASK_PORT% ===
                    for /f "tokens=2 delims== " %%P in ('wmic process call create "cmd /c set FLASK_APP=app\\api.py ^& flask run --host=127.0.0.1 --port=%FLASK_PORT% ^> flask.log 2^>^&1" ^| find "ProcessId"') do (
                        echo %%P > %FLASK_PID_FILE%
                    )

                    echo === Starting WireMock on %WIREMOCK_PORT% ===
                    if exist "%WIREMOCK_JAR%" (
                        for /f "tokens=2 delims== " %%P in ('wmic process call create "cmd /c java -jar %WIREMOCK_JAR% --port %WIREMOCK_PORT% --root-dir %WIREMOCK_DIR% ^> wiremock.log 2^>^&1" ^| find "ProcessId"') do (
                            echo %%P > %WIREMOCK_PID_FILE%
                        )
                    ) else (
                        echo Wiremock jar not found at %WIREMOCK_JAR%
                        exit /b 2
                    )

                    echo === Waiting for ports (timeout %WAIT_TIMEOUT_SECONDS%s) ===
                    python -c "import socket,time,sys; t=time.time()+int('%WAIT_TIMEOUT_SECONDS%'); \
def ok(p): \
  try: s=socket.create_connection(('127.0.0.1',p),1); s.close(); return True; \
  except OSError: return False; \
ports=[int('%FLASK_PORT%'),int('%WIREMOCK_PORT%')]; \
while time.time()<t and not all(ok(p) for p in ports): time.sleep(0.5); \
print('Ports ready:', {p:ok(p) for p in ports}); \
sys.exit(0 if all(ok(p) for p in ports) else 1)"
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
                echo === Cleaning up services (PID-based) ===

                if exist %WIREMOCK_PID_FILE% (
                    set /p WMPID=<%WIREMOCK_PID_FILE%
                    echo Stopping WireMock PID %WMPID%
                    taskkill /F /PID %WMPID% >NUL 2>&1
                    del /f /q %WIREMOCK_PID_FILE% >NUL 2>&1
                ) else (
                    echo WireMock PID file not found, skipping
                )

                if exist %FLASK_PID_FILE% (
                    set /p FPID=<%FLASK_PID_FILE%
                    echo Stopping Flask PID %FPID%
                    taskkill /F /PID %FPID% >NUL 2>&1
                    del /f /q %FLASK_PID_FILE% >NUL 2>&1
                ) else (
                    echo Flask PID file not found, skipping
                )

                echo === Logs (if any) ===
                if exist flask.log type flask.log
                if exist wiremock.log type wiremock.log
            '''
            archiveArtifacts artifacts: 'reports/*.xml,*.log,*.pid', fingerprint: true, allowEmptyArchive: true
        }
    }
}
