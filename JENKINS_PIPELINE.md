# Jenkins Pipeline 说明文档

## 📋 概述

这个 Jenkins Pipeline 为 Android Architecture Samples 项目自动化了完整的构建、测试和打包流程。

---

## 🏗️ Pipeline 结构

### 核心阶段 (Stages)

```
Checkout → Environment Setup → Lint → Build → Unit Tests → Build APK/AAB → Analysis & Reports
```

---

## 📌 详细说明

### 1️⃣ **Checkout 阶段**
- **功能**: 从 Git 仓库检出代码
- **操作**:
  - 克隆或更新代码
  - 提取 Git 元信息（提交信息、作者、提交哈希）
  - 用于追踪哪个开发者的代码导致了构建结果
  
**示例输出**:
```
✅ 提交信息: Fix: 修复登录功能
✅ 提交作者: John Doe
✅ 提交哈希: a1b2c3d
```

---

### 2️⃣ **Environment Setup 阶段**
- **功能**: 验证构建环境
- **检查项**:
  - ✅ JDK 版本
  - ✅ Gradle 版本
  - ✅ Android SDK 是否可用
  
**为什么重要**: 确保所有构建工具都正确安装和配置

---

### 3️⃣ **Code Quality - Lint 阶段** ⚙️ *可选*
- **条件**: 仅当 `RUN_LINT = true` 时执行
- **功能**: 运行 Android Lint 代码检查
- **检查内容**:
  - 未使用的资源
  - 性能问题
  - 安全问题
  - API 兼容性问题

**输出**: `lint-results.xml` 报告文件

---

### 4️⃣ **Build 阶段** 🔨
- **功能**: 编译项目代码
- **参数**:
  - `BUILD_VARIANT`: `debug` 或 `release`
  - 使用并行编译优化速度 (`-Dorg.gradle.parallel=true`)
- **输出**: 编译的中间文件

**命令示例**:
```bash
./gradlew clean assemble[Debug|Release] --stacktrace
```

---

### 5️⃣ **Unit Tests 阶段** 🧪 *可选*
- **条件**: 仅当 `RUN_TESTS = true` 时执行
- **功能**: 运行项目中的所有单元测试
  - app 模块的单元测试
  - shared-test 模块的共享测试
  
- **输出**:
  - `test-results/**/*.xml` - 测试结果 XML
  - HTML 测试报告

**测试框架**: Espresso (instrumented) + JUnit (unit)

---

### 6️⃣ **Build APK/AAB 阶段** 📦
- **Debug 构建**:
  - 生成 APK 文件
  - 可直接在模拟器/真机安装和运行
  
- **Release 构建**:
  - 生成 AAB (Android App Bundle)
  - **需要签名配置** (密钥库文件)
  - 用于 Google Play Store 提交

**输出位置**: `app/build/outputs/`

---

### 7️⃣ **Analysis & Reports 阶段** 📊
- **生成的报告**:
  - 项目结构报告
  - 依赖关系图
  - 生成的制件列表
  
- **对应的文件**:
  ```
  app/build/outputs/
  ├── apk/              # APK 文件
  ├── bundle/           # AAB 文件
  └── logs/             # 构建日志
  ```

---

## ⚙️ 配置参数

### 构建参数
Pipeline 支持以下可配置参数:

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `BUILD_VARIANT` | Choice | debug | 构建类型 (debug/release) |
| `RUN_TESTS` | Boolean | true | 是否运行单元测试 |
| `RUN_LINT` | Boolean | true | 是否运行 Lint 检查 |

### 环境变量

```groovy
ANDROID_SDK_ROOT = '/usr/local/android-sdk'
ANDROID_HOME = '/usr/local/android-sdk'
GRADLE_OPTS = '-Xmx2048m -Dfile.encoding=UTF-8'
```

**这些需要根据你的 Jenkins 服务器环境修改**

---

## 🔧 前置要求

### Jenkins 中需要安装的插件

```
1. Android Lint Plugin
2. JUnit Plugin (通常已装)
3. HTML Publisher Plugin
4. Timestamper Plugin
5. Log Parser Plugin (可选)
6. Slack Notification Plugin (可选)
```

### 系统要求

```
1. Android SDK (API 34+)
2. JDK 11+
3. Gradle 8.0+
4. 4GB+ RAM (推荐)
5. Linux/macOS/Windows
```

### 配置步骤

```bash
# 1. 在 Jenkins 系统配置中设置 Android SDK 路径
# Jenkins 管理 → 系统配置 → ANDROID_HOME=/path/to/android-sdk

# 2. 如果使用 Release 构建，配置签名密钥
# 将密钥库文件放在安全位置，配置到 build.gradle.kts

# 3. (可选) 配置 Slack 通知
# Jenkins 管理 → 系统配置 → Slack → 填入 Token 和 Channel
```

---

## 📊 Pipeline Options (管道选项)

### 已配置的选项:

| 选项 | 值 | 说明 |
|------|-----|------|
| buildDiscarder | 30 builds | 保留最近 30 个构建历史 |
| timeout | 30 minutes | 构建超时时间 |
| timestamps | ✅ | 为日志添加时间戳 |
| disableConcurrentBuilds | ✅ | 禁止同时运行多个构建 |

**优点**:
- 节省磁盘空间
- 防止资源竞争
- 便于日志查阅

---

## 🚀 执行流程示例

### Debug 构建流程:

```
1. [2 min]  Checkout → 获取代码
2. [1 min]  Environment Setup → 验证环境
3. [3 min]  Lint → 代码检查
4. [5 min]  Build → 编译代码
5. [4 min]  Unit Tests → 运行单元测试
6. [2 min]  Build APK → 生成 APK
7. [1 min]  Analysis → 生成报告
───────────────────────────────
    总耗时: ~18 分钟
```

### Release 构建流程 (相似，但需要签名):

```
类似，但多出签名步骤
总耗时: ~20 分钟
```

---

## 📈 输出制件

### Debug 构建输出:

```
app/build/outputs/apk/debug/
├── app-debug.apk                 # 调试版 APK
└── output-metadata.json          # 元数据
```

### Release 构建输出:

```
app/build/outputs/bundle/release/
├── app-release.aab               # Android App Bundle
└── BundleConfig.pb               # 配置信息
```

### 测试报告输出:

```
app/build/
├── reports/tests/                # 单元测试报告 HTML
├── test-results/                 # JUnit XML 结果
└── outputs/code-coverage/        # 代码覆盖率 (如果启用)
```

---

## ⚠️ 常见问题

### Q1: 构建失败 "ANDROID_HOME not found"

**解决方案**:
```bash
# 在 Jenkins 执行节点上设置
export ANDROID_HOME=/path/to/android-sdk
# 或在 Jenkinsfile 中修改环境变量路径
```

### Q2: Lint 报告找不到

**原因**: 使用了不同的构建变体
**解决方案**: 修改 Lint 模式匹配
```groovy
pattern: '**/lint-results-*.xml'  // 支持多个文件
```

### Q3: 如何跳过某个阶段?

**方法**:
```groovy
// 1. 通过参数 (已支持)
when { expression { params.RUN_TESTS == true } }

// 2. 通过条件
when { branch 'main' }  // 仅在 main 分支运行某些阶段
```

### Q4: 测试失败构建是否会失败?

**默认行为**: 是的，任何失败都会导致构建失败

**如果想继续**: 
```groovy
sh './gradlew test --continue'  // 继续运行其他测试
```

---

## 🔐 安全考虑

### Release 构建签名配置

```groovy
// 不要在代码中硬编码密钥!
// 使用 Jenkins Credentials

withCredentials([file(credentialsId: 'keystore-file', variable: 'KEYSTORE')]) {
    sh '''
        ./gradlew -Pandroid.injected.signing.store.file=$KEYSTORE \
                 -Pandroid.injected.signing.store.password=$KEYSTORE_PASS \
                 build
    '''
}
```

### 敏感信息处理

- ✅ 使用 Jenkins Credentials Store
- ✅ 不要在 Jenkinsfile 中暴露密钥
- ✅ 使用 masked passwords
- ✅ 定期轮换密钥

---

## 📝 日志分析

### 构建成功日志示例:

```
✅ 提交信息: Feat: 添加新功能
✅ 提交作者: Jane Smith
✅ 提交哈希: b2c3d4e
JDK 版本: openjdk 11.0.15
Gradle 版本: 8.0.1
🔍 运行代码检查 (Lint)...
  > Lint 完成，未发现严重问题
🔨 构建 debug 版本...
  > BUILD SUCCESSFUL
🧪 运行单元测试...
  > Test passed: 125 tests
📦 打包 debug APK...
  > Generated app-debug.apk
✅ 构建成功!
```

### 构建失败日志示例:

```
❌ 构建失败!
Compilation failed:
  > error: cannot find symbol 'myVariable'
    Location: MainActivity.kt:42
    
查看完整堆栈跟踪以获取详情
```

---

## 🔄 Pipeline 定时执行

### 配置定时构建

```groovy
// 在 Jenkinsfile 顶部添加
triggers {
    pollSCM('H H * * *')  // 每天凌晨运行
    // 或使用 cron 语法: '0 2 * * *'
}
```

### 触发器选项:

```groovy
triggers {
    // 每小时检查一次代码变化
    pollSCM('H * * * *')
    
    // GitHub webhook 自动触发
    githubPush()
    
    // 每天早上 2 点运行
    cron('0 2 * * *')
}
```

---

## 📢 通知配置 (可选)

### Slack 通知示例:

```groovy
post {
    success {
        slackSend(
            color: 'good',
            message: "✅ 构建成功: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n${env.BUILD_URL}"
        )
    }
    failure {
        slackSend(
            color: 'danger',
            message: "❌ 构建失败: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n${env.BUILD_URL}"
        )
    }
}
```

### Email 通知示例:

```groovy
post {
    failure {
        emailext(
            subject: "构建失败: ${env.JOB_NAME}",
            body: "请查看: ${env.BUILD_URL}",
            to: "${env.CHANGE_AUTHOR_EMAIL}"
        )
    }
}
```

---

## 🎯 最佳实践

### ✅ 推荐做法

1. **定期清理构建历史** (已配置: 30 builds)
2. **使用并行构建加速** (已配置: `-Dorg.gradle.parallel=true`)
3. **记录详细日志** (已配置: `--stacktrace`)
4. **分离 Debug 和 Release** (已支持: `BUILD_VARIANT` 参数)
5. **自动备份制件** (已配置: `archiveArtifacts`)

### ❌ 避免做法

1. ❌ 在构建中存储密钥 → 使用 Jenkins Credentials
2. ❌ 跳过测试以加快构建 → 保持代码质量
3. ❌ 忽视 Lint 警告 → 预防未来问题
4. ❌ 手动管理制件 → 使用自动归档
5. ❌ 构建所有 git 分支 → 使用分支过滤器

---

## 🚀 快速开始

### 1. 在 Jenkins 中新建 Pipeline 任务

```
1. Jenkins → 新建项目 → Pipeline
2. 项目名: "architecture-samples"
3. 定义: "Pipeline script from SCM"
4. SCM: Git
5. 仓库地址: https://github.com/android/architecture-samples.git
6. 脚本路径: Jenkinsfile
```

### 2. 配置系统环境

```bash
# Jenkins 执行节点上
export ANDROID_HOME=/usr/local/android-sdk
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk
```

### 3. 运行首次构建

```
点击 "Build with Parameters" → 选择参数 → "开始构建"
```

### 4. 查看结果

```
构建完成后:
- 查看 "Console Output" 查看日志
- 查看 "Test Report" 查看测试结果
- 查看 "Artifacts" 下载生成的 APK
```

---

## 📚 相关资源

- [Jenkins Pipeline 文档](https://www.jenkins.io/doc/book/pipeline/)
- [Android 构建系统文档](https://developer.android.com/build)
- [Gradle 官方文档](https://gradle.org/docs/)
- [Android Lint 检查列表](https://developer.android.com/studio/write/lint)

---

## 💡 常用命令

```bash
# 仅构建，不运行测试
./gradlew assemble[Debug|Release]

# 运行单元测试
./gradlew test[Debug|Release]

# 运行 Lint 检查
./gradlew lint[Debug|Release]

# 构建并生成测试报告
./gradlew build --continue

# 清理构建缓存
./gradlew clean

# 查看依赖树
./gradlew dependencies
```

---

## 📞 支持与反馈

有问题或建议? 请提交 Issue 或 PR 到 GitHub 仓库。

