pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        VENV_DIR = ".venv"
        FLASK_PORT = "5000"
        WIREMOCK_PORT = "9090"
        WIREMOCK_JAR = "test\\wiremock\\wiremock-standalone-3.13.2.jar"
        WAIT_TIMEOUT_SECONDS = "120"
        FLASK_PID_FILE = "flask.pid"
        WIREMOCK_PID_FILE = "wiremock.pid"
        FLASK_LOG_OUT = "flask.out.log"
        FLASK_LOG_ERR = "flask.err.log"
        WIREMOCK_LOG_OUT = "wiremock.out.log"
        WIREMOCK_LOG_ERR = "wiremock.err.log"
        WAIT_SCRIPT = "wait_ports.py"
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

                    echo === Pre-clean old PID/log files (workspace leftovers) ===
                    del /f /q %FLASK_PID_FILE% 1>NUL 2>&1
                    del /f /q %WIREMOCK_PID_FILE% 1>NUL 2>&1
                    del /f /q %FLASK_LOG_OUT% 1>NUL 2>&1
                    del /f /q %FLASK_LOG_ERR% 1>NUL 2>&1
                    del /f /q %WIREMOCK_LOG_OUT% 1>NUL 2>&1
                    del /f /q %WIREMOCK_LOG_ERR% 1>NUL 2>&1
                    del /f /q %WAIT_SCRIPT% 1>NUL 2>&1

                    echo === Starting Flask on %FLASK_PORT% ===
                    powershell -NoProfile -ExecutionPolicy Bypass -Command ^
                      "$p = Start-Process -FilePath 'python' -ArgumentList '-m','flask','--app','app.api','run','--host','127.0.0.1','--port','%FLASK_PORT%' -PassThru -RedirectStandardOutput '%FLASK_LOG_OUT%' -RedirectStandardError '%FLASK_LOG_ERR%';" ^
                      "$p.Id | Out-File -Encoding ascii '%FLASK_PID_FILE%';" ^
                      "Write-Host ('Flask PID: ' + $p.Id)"

                    echo === Starting WireMock on %WIREMOCK_PORT% ===
                    if not exist "%WIREMOCK_JAR%" (
                        echo ERROR: WireMock jar not found at %WIREMOCK_JAR%
                        exit /b 2
                    )

                    powershell -NoProfile -ExecutionPolicy Bypass -Command ^
                     "$p = Start-Process -FilePath 'java.exe' -ArgumentList '-jar','%WIREMOCK_JAR%','--port','%WIREMOCK_PORT%','--root-dir','test\\wiremock' -PassThru -RedirectStandardOutput '%WIREMOCK_LOG_OUT%' -RedirectStandardError '%WIREMOCK_LOG_ERR%';" ^
                     "$p.Id | Out-File -Encoding ascii '%WIREMOCK_PID_FILE%';" ^
                     "Write-Host ('WireMock PID: ' + $p.Id)"

                    echo === Small grace period (avoid race) ===
                    timeout /t 2 /nobreak >NUL

                    echo === Writing wait script (%WAIT_SCRIPT%) ===
                    >  %WAIT_SCRIPT% echo import socket, time, sys
                    >> %WAIT_SCRIPT% echo PORTS = [int("%FLASK_PORT%"), int("%WIREMOCK_PORT%")]
                    >> %WAIT_SCRIPT% echo TIMEOUT = int("%WAIT_TIMEOUT_SECONDS%")
                    >> %WAIT_SCRIPT% echo def open_port(p):
                    >> %WAIT_SCRIPT% echo^    try:
                    >> %WAIT_SCRIPT% echo^        s = socket.create_connection(("127.0.0.1", p), 1)
                    >> %WAIT_SCRIPT% echo^        s.close()
                    >> %WAIT_SCRIPT% echo^        return True
                    >> %WAIT_SCRIPT% echo^    except OSError:
                    >> %WAIT_SCRIPT% echo^        return False
                    >> %WAIT_SCRIPT% echo end = time.time() + TIMEOUT
                    >> %WAIT_SCRIPT% echo while time.time() ^< end:
                    >> %WAIT_SCRIPT% echo^    states = {p: open_port(p) for p in PORTS}
                    >> %WAIT_SCRIPT% echo^    if all(states.values()):
                    >> %WAIT_SCRIPT% echo^        print("Ports ready:", states)
                    >> %WAIT_SCRIPT% echo^        sys.exit(0)
                    >> %WAIT_SCRIPT% echo^    time.sleep(0.5)
                    >> %WAIT_SCRIPT% echo print("Ports NOT ready:", {p: open_port(p) for p in PORTS})
                    >> %WAIT_SCRIPT% echo sys.exit(1)

                    echo === Waiting for ports (timeout %WAIT_TIMEOUT_SECONDS%s) ===
                    py -3 %WAIT_SCRIPT%
                    set WAIT_EXIT=%ERRORLEVEL%

                    echo === Diagnostics: netstat for ports ===
                    netstat -ano | findstr ":%FLASK_PORT%"
                    netstat -ano | findstr ":%WIREMOCK_PORT%"

                    if not "%WAIT_EXIT%"=="0" (
                        echo === Wait failed. Dumping logs ===
                        if exist %FLASK_LOG_OUT% type %FLASK_LOG_OUT%
                        if exist %FLASK_LOG_ERR% type %FLASK_LOG_ERR%
                        if exist %WIREMOCK_LOG_OUT% type %WIREMOCK_LOG_OUT%
                        if exist %WIREMOCK_LOG_ERR% type %WIREMOCK_LOG_ERR%
                        exit /b 1
                    )
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
                echo === Cleaning up services (PID-based, PowerShell) ===
                powershell -NoProfile -ExecutionPolicy Bypass -Command ^
                  "$ErrorActionPreference='SilentlyContinue';" ^
                  "if (Test-Path '%WIREMOCK_PID_FILE%') { $pid=(Get-Content '%WIREMOCK_PID_FILE%'|Select-Object -First 1); if ($pid) { Stop-Process -Id $pid -Force -ErrorAction SilentlyContinue; Write-Host ('Stopped WireMock PID ' + $pid) } Remove-Item -Force '%WIREMOCK_PID_FILE%' }" ^
                  "if (Test-Path '%FLASK_PID_FILE%') { $pid=(Get-Content '%FLASK_PID_FILE%'|Select-Object -First 1); if ($pid) { Stop-Process -Id $pid -Force -ErrorAction SilentlyContinue; Write-Host ('Stopped Flask PID ' + $pid) } Remove-Item -Force '%FLASK_PID_FILE%' }"

                echo === Logs (if any) ===
                if exist %FLASK_LOG_OUT% type %FLASK_LOG_OUT%
                if exist %FLASK_LOG_ERR% type %FLASK_LOG_ERR%
                if exist %WIREMOCK_LOG_OUT% type %WIREMOCK_LOG_OUT%
                if exist %WIREMOCK_LOG_ERR% type %WIREMOCK_LOG_ERR%
            '''
            archiveArtifacts artifacts: 'reports/*.xml,*.log,*.pid,*.py', fingerprint: true, allowEmptyArchive: true
        }
    }
}
