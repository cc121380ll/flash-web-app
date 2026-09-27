pipeline {
    agent any

    environment {
        // 按自己机器上的 Python 路径修改；本次已装官方 Python 3.13.15 到该目录
        PYTHON_PATH = 'D:\\install\\Python313\\python.exe'

        // 修复（教程里缺这一行）：Jenkins 在构建结束时会杀掉本次构建启动的所有子进程，
        // 包括 Deploy 阶段用 start 拉起来的 Flask。设成 dontKillMe 才能让服务活着，
        // 否则表现为「构建成功但服务起不来」。
        BUILD_ID = 'dontKillMe'

        // 修复（本机实测踩到）：Python 默认会加载用户级 site-packages
        // (C:\Users\<你>\AppData\Roaming\Python\Python313) 里的第三方包。
        // 那里面有一份不完整的环境（langsmith 等），pytest 一启动就加载它的
        // pytest11 插件，报 ModuleNotFoundError: No module named 'httpx'，
        // 导致 Test 阶段直接失败、htmlcov 不生成。
        // 置 1 让流水线只看 D:\install\Python313 自己的 site-packages。
        PYTHONNOUSERSITE = '1'
    }

    stages {
        stage('Verify Python Path') {
            steps {
                bat """
                    @echo off
                    echo "=== 验证Python313路径及版本 ==="
                    ${PYTHON_PATH} --version
                    echo "Python路径验证通过！"
                """
            }
        }

        stage('Checkout') {
            steps {
                // TODO: 改成你自己的仓库（HTTPS 格式）
                git url: 'https://github.com/your_github_username/flash-web-app.git', branch: 'main'
            }
        }

        stage('Fix pip') {
            steps {
                bat """
                    @echo off
                    echo "=== 修复Python313的pip环境 ==="
                    ${PYTHON_PATH} -m ensurepip --upgrade
                    ${PYTHON_PATH} -m pip --version
                """
            }
        }

        stage('Install Dependencies') {
            steps {
                bat """
                    @echo off
                    ${PYTHON_PATH} -m pip install --upgrade pip
                    ${PYTHON_PATH} -m pip install -r requirements.txt
                """
            }
        }

        stage('Lint') {
            steps {
                bat "${PYTHON_PATH} -m pip install flake8 && ${PYTHON_PATH} -m flake8 app.py tests/"
            }
        }

        stage('Test') {
            steps {
                bat """
                    ${PYTHON_PATH} -m pip install pytest pytest-cov
                    ${PYTHON_PATH} -m pytest --cov=app tests/ --cov-report=html
                """
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: 'htmlcov',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('Build') {
            steps {
                bat """
                    ${PYTHON_PATH} -m pip install pyinstaller
                    ${PYTHON_PATH} -m PyInstaller --onefile app.py
                """
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying application...'
                // 注意：重复构建时 5000 端口仍被上一次的实例占用，会报 Address already in use，
                // 属于正常现象（老实例还在跑）；想重启请先结束占用 5000 端口的进程。
                bat "start ${PYTHON_PATH} app.py"
            }
        }
    }

    post {
        success { echo 'CI/CD pipeline completed successfully!' }
        failure { echo 'CI/CD pipeline failed!' }
    }
}
