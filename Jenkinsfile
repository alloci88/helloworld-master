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
        WIREMOCK_JAR = "test\\wiremock\\wiremock-standalone-3.13.2.jar"

        WAIT_TIMEOUT_SECONDS = "60"

        FLASK_PID_FILE = "flask.pid"
        WIREMOCK_PID_FILE = "wiremock.pid"

        FLASK_LOG = "flask.log"
        WIREMOCK_LOG = "wiremock.log"
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
        
                    echo === Starting Flask on %FLASK_PORT% ===
                    powershell -NoProfile -ExecutionPolicy Bypass -Command ^
                      "$ErrorActionPreference='Stop';" ^
                      "if (Test-Path '%FLASK_PID_FILE%') { Remove-Item -Force '%FLASK_PID_FILE%' }" ^
                      "if (Test-Path '%FLASK_LOG%') { Remove-Item -Force '%FLASK_LOG%' }" ^
                      "$p = Start-Process -FilePath 'cmd.exe' -ArgumentList '/c','call %VENV_DIR%\\Scripts\\activate.bat ^& set PYTHONPATH=%CD% ^& set FLASK_RUN_PORT=%FLASK_PORT% ^& python -m flask --app app.api run --host 127.0.0.1 --port %FLASK_PORT% > %FLASK_LOG% 2>&1' -PassThru -WindowStyle Hidden;" ^
                      "$p.Id | Out-File -Encoding ascii '%FLASK_PID_FILE%';" ^
                      "Write-Host ('Flask PID: ' + $p.Id)"
        
                    echo === Starting WireMock on %WIREMOCK_PORT% ===
                    powershell -NoProfile -ExecutionPolicy Bypass -Command ^
                      "$ErrorActionPreference='Stop';" ^
                      "if (!(Test-Path '%WIREMOCK_JAR%')) { Write-Host 'WireMock jar not found'; exit 2 }" ^
                      "if (Test-Path '%WIREMOCK_PID_FILE%') { Remove-Item -Force '%WIREMOCK_PID_FILE%' }" ^
                      "if (Test-Path '%WIREMOCK_LOG%') { Remove-Item -Force '%WIREMOCK_LOG%' }" ^
                      "$p = Start-Process -FilePath 'java.exe' -ArgumentList '-jar','%WIREMOCK_JAR%','--port','%WIREMOCK_PORT%','--root-dir','%WIREMOCK_DIR%' -PassThru -WindowStyle Hidden -RedirectStandardOutput '%WIREMOCK_LOG%';" ^
                      "$p.Id | Out-File -Encoding ascii '%WIREMOCK_PID_FILE%';" ^
                      "Write-Host ('WireMock PID: ' + $p.Id)"
        
                    echo === Waiting for ports (timeout %WAIT_TIMEOUT_SECONDS%s) ===
                    powershell -NoProfile -ExecutionPolicy Bypass -Command ^
                      "$timeout=[int]'%WAIT_TIMEOUT_SECONDS%';" ^
                      "$ports=@([int]'%FLASK_PORT%',[int]'%WIREMOCK_PORT%');" ^
                      "$deadline=(Get-Date).AddSeconds($timeout);" ^
                      "function Test-Port($p){ try { $c=New-Object Net.Sockets.TcpClient; $c.Connect('127.0.0.1',$p); $c.Close(); $true } catch { $false } }" ^
                      "while((Get-Date) -lt $deadline) {" ^
                      "  $states = $ports | ForEach-Object { Test-Port $_ };" ^
                      "  if(($states | Where-Object {$_ -eq $false}).Count -eq 0) { Write-Host 'Ports ready'; exit 0 }" ^
                      "  Start-Sleep -Milliseconds 500" ^
                      "}" ^
                      "Write-Host 'Ports NOT ready'; exit 1"
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
                  "if (Test-Path '%WIREMOCK_PID_FILE%') { $pid = (Get-Content '%WIREMOCK_PID_FILE%' | Select-Object -First 1); if ($pid) { Stop-Process -Id $pid -Force -ErrorAction SilentlyContinue; Write-Host ('Stopped WireMock PID ' + $pid) } Remove-Item -Force '%WIREMOCK_PID_FILE%' } else { Write-Host 'WireMock PID file not found, skipping' }" ^
                  "if (Test-Path '%FLASK_PID_FILE%') { $pid = (Get-Content '%FLASK_PID_FILE%' | Select-Object -First 1); if ($pid) { Stop-Process -Id $pid -Force -ErrorAction SilentlyContinue; Write-Host ('Stopped Flask PID ' + $pid) } Remove-Item -Force '%FLASK_PID_FILE%' } else { Write-Host 'Flask PID file not found, skipping' }"

                echo === Logs (if any) ===
                if exist %FLASK_LOG% type %FLASK_LOG%
                if exist %WIREMOCK_LOG% type %WIREMOCK_LOG%
                exit /b 0
            '''
            archiveArtifacts artifacts: 'reports/*.xml,*.log,*.pid', fingerprint: true, allowEmptyArchive: true   
        }
    }
}
