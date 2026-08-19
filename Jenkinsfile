pipeline {
    agent any

    options {
        // 保留最近30个构建的历史
        buildDiscarder(logRotator(numToKeepStr: '30'))
        // 超时设置为30分钟
        timeout(time: 30, unit: 'MINUTES')
        // 加上时间戳到控制台输出
        timestamps()
        // 禁用并发构建
        disableConcurrentBuilds()
    }

    parameters {
        choice(
            name: 'BUILD_VARIANT',
            choices: ['debug', 'release'],
            description: '选择构建类型'
        )
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: '是否运行单元测试'
        )
        booleanParam(
            name: 'RUN_LINT',
            defaultValue: true,
            description: '是否运行代码检查'
        )
    }

    environment {
        // Android SDK 路径
        ANDROID_SDK_ROOT = '/usr/local/android-sdk'
        ANDROID_HOME = '/usr/local/android-sdk'
        PATH = "${ANDROID_HOME}/tools:${ANDROID_HOME}/tools/bin:${ANDROID_HOME}/platform-tools:${PATH}"
        // 构建工具版本
        GRADLE_OPTS = '-Xmx2048m -Dfile.encoding=UTF-8'
    }

    stages {
        stage('Checkout') {
            steps {
                echo '📥 检出代码...'
                sh '''
                    cd /Users/willguo/projects/architecture-samples
                '''
                script {
                    // 获取 Git 信息
                    env.GIT_COMMIT_MSG = sh(
                        script: "cd /Users/willguo/projects/architecture-samples && git log -1 --pretty=%B",
                        returnStdout: true
                    ).trim()
                    env.GIT_AUTHOR = sh(
                        script: "cd /Users/willguo/projects/architecture-samples && git log -1 --pretty=%an",
                        returnStdout: true
                    ).trim()
                    env.BUILD_NUMBER_CUSTOM = sh(
                        script: "cd /Users/willguo/projects/architecture-samples && git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                }
                echo "✅ 提交信息: ${env.GIT_COMMIT_MSG}"
                echo "✅ 提交作者: ${env.GIT_AUTHOR}"
                echo "✅ 提交哈希: ${env.BUILD_NUMBER_CUSTOM}"
            }
        }

        stage('Environment Setup') {
            steps {
                echo '⚙️ 环境检查...'
                sh '''
                    echo "JDK 版本:"
                    java -version
                    echo "Gradle 版本:"
                    ./gradlew --version
                    echo "Android SDK 检查:"
                    ls -la $ANDROID_HOME 2>/dev/null || echo "Android SDK 未找到"
                '''
            }
        }

        stage('Code Quality - Lint') {
            when {
                expression { params.RUN_LINT == true }
            }
            steps {
                echo '🔍 运行代码检查 (Lint)...'
                sh '''
                    cd /Users/willguo/projects/architecture-samples
                    ./gradlew lint${BUILD_VARIANT.capitalize()} --stacktrace || true
                '''
                // 收集 lint 报告
                androidLint canComputeNew: false, defaultEncoding: '', healthy: '', pattern: '**/lint-results*.xml', unHealthy: '', unstableThreshold: '0'
            }
        }

        stage('Build') {
            steps {
                echo "🔨 构建 ${params.BUILD_VARIANT} 版本..."
                sh '''
                    cd /Users/willguo/projects/architecture-samples
                    ./gradlew clean assemble${BUILD_VARIANT.capitalize()} \
                        -Dorg.gradle.parallel=true \
                        -Dorg.gradle.workers.max=4 \
                        --stacktrace
                '''
            }
        }

        stage('Unit Tests') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                echo '🧪 运行单元测试...'
                sh '''
                    cd /Users/willguo/projects/architecture-samples
                    ./gradlew test${BUILD_VARIANT.capitalize()} \
                        --stacktrace \
                        -Dorg.gradle.parallel=true
                '''
            }
            post {
                always {
                    // 发布测试报告
                    junit '**/build/test-results/**/*.xml'
                    // 收集代码覆盖率报告 (如果生成了)
                    publishHTML target: [
                        reportDir: 'app/build/reports/tests',
                        reportFiles: 'index.html',
                        reportName: '单元测试报告'
                    ] || true
                }
            }
        }

        stage('Build APK/AAB') {
            steps {
                echo "📦 打包 ${params.BUILD_VARIANT} APK..."
                sh '''
                    cd /Users/willguo/projects/architecture-samples
                    if [ "${BUILD_VARIANT}" == "release" ]; then
                        # Release 构建需要签名配置
                        echo "⚠️ Release 构建需要配置签名密钥"
                        ./gradlew bundle${BUILD_VARIANT.capitalize()} --stacktrace || true
                    else
                        # Debug 构建
                        ./gradlew assemble${BUILD_VARIANT.capitalize()} --stacktrace
                    fi
                '''
            }
        }

        stage('Analysis & Reports') {
            steps {
                echo '📊 生成分析报告...'
                sh '''
                    cd /Users/willguo/projects/architecture-samples
                    # 生成构建报告
                    ./gradlew projectReport --stacktrace || true
                    
                    # 获取 APK 信息
                    echo "生成的制件:"
                    find app/build -name "*.apk" -o -name "*.aab" | head -10
                '''
            }
        }
    }

    post {
        always {
            echo '🧹 清理工作...'
            // 保存构建日志
            sh 'echo "构建完成于: $(date)" > build.log'
            
            // 归档制件
            script {
                try {
                    archiveArtifacts artifacts: 'app/build/outputs/**/*.apk,app/build/outputs/**/*.aab', 
                        allowEmptyArchive: true
                } catch (Exception e) {
                    echo "⚠️ 未找到制件文件"
                }
            }
        }

        success {
            echo '✅ 构建成功!'
            // 可以集成通知，例如 Slack、Email 等
            // slackSend(color: 'good', message: "构建成功: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }

        failure {
            echo '❌ 构建失败!'
            // slackSend(color: 'danger', message: "构建失败: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }

        unstable {
            echo '⚠️ 构建不稳定 (有测试失败)'
            // slackSend(color: 'warning', message: "构建不稳定: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }

        cleanup {
            echo '🗑️ 清理工作空间...'
            // 可选：清理工作空间
            // deleteDir()
        }
    }
}
