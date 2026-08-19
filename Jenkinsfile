pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
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
        GRADLE_OPTS = '-Xmx2048m -Dfile.encoding=UTF-8'
        BUILD_VARIANT_UPPER = "${params.BUILD_VARIANT.capitalize()}"
    }

    stages {
        stage('Setup') {
            steps {
                echo '⚙️ 设置构建环境...'
                sh '''
                    echo "sdk.dir=$HOME/Library/Android/sdk" > local.properties
                    cat local.properties
                '''
            }
        }

        stage('Checkout') {
            steps {
                echo '📥 检出代码...'
                script {
                    env.GIT_COMMIT_MSG = sh(
                        script: "git log -1 --pretty=%B",
                        returnStdout: true
                    ).trim()
                    env.GIT_AUTHOR = sh(
                        script: "git log -1 --pretty=%an",
                        returnStdout: true
                    ).trim()
                    env.GIT_HASH = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                }
                echo "✅ 提交信息: ${env.GIT_COMMIT_MSG}"
                echo "✅ 提交作者: ${env.GIT_AUTHOR}"
                echo "✅ 提交哈希: ${env.GIT_HASH}"
            }
        }

        stage('Environment Setup') {
            steps {
                echo '🔍 环境检查...'
                sh """
                    echo "JDK 版本:"
                    java -version
                    echo "Gradle 版本:"
                    ./gradlew --version
                """
            }
        }

        stage('Code Quality - Lint') {
            when {
                expression { params.RUN_LINT == true }
            }
            steps {
                echo "🔍 运行代码检查 (Lint) - ${BUILD_VARIANT_UPPER}..."
                sh """
                    ./gradlew lint${BUILD_VARIANT_UPPER} --stacktrace || true
                """
            }
        }

        stage('Build') {
            steps {
                echo "🔨 构建 ${params.BUILD_VARIANT} 版本..."
                sh """
                    ./gradlew clean assemble${BUILD_VARIANT_UPPER} \
                        -Dorg.gradle.parallel=true \
                        -Dorg.gradle.workers.max=4 \
                        --stacktrace
                """
            }
        }

        stage('Unit Tests') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                echo "🧪 运行单元测试 - ${BUILD_VARIANT_UPPER}..."
                sh """
                    ./gradlew test${BUILD_VARIANT_UPPER} \
                        --stacktrace \
                        -Dorg.gradle.parallel=true || true
                """
            }
            post {
                always {
                    junit testResults: '**/build/test-results/**/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('Build APK/AAB') {
            steps {
                echo "📦 打包 ${params.BUILD_VARIANT} 版本..."
                sh """
                    if [ "${params.BUILD_VARIANT}" = "release" ]; then
                        ./gradlew bundle${BUILD_VARIANT_UPPER} --stacktrace || true
                    else
                        ./gradlew assemble${BUILD_VARIANT_UPPER} --stacktrace
                    fi
                """
            }
        }

        stage('Analysis & Reports') {
            steps {
                echo '📊 生成分析报告...'
                sh """
                    ./gradlew projectReport --stacktrace || true
                    echo "生成的制件:"
                    find app/build -name "*.apk" -o -name "*.aab" 2>/dev/null | head -10
                """
            }
        }
    }

    post {
        always {
            echo '🧹 清理工作...'
            sh 'echo "构建完成于: $(date)" > build.log'
            
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
        }

        failure {
            echo '❌ 构建失败!'
        }

        unstable {
            echo '⚠️ 构建不稳定'
        }
    }
}
