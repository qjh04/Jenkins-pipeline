pipeline {
    agent any

    environment {
        GOROOT = '/usr/local/go'
        GOPATH = '/var/lib/jenkins/workspace/go'
        PATH = "${GOROOT}/bin:${PATH}"
        // REMOTE_HOST = '115.190.24.62'
        REMOTE_HOST = '192.168.0.69'
        REMOTE_USER = 'root'
        // REMOTE_DIR = '/export/App/wonderLand_go'
        // TEMP_DIR = '/export/App/wonderLand_go_temp'
        REMOTE_DIR = '/export/temp/wonderLand_go'
        TEMP_DIR = '/export/temp/wonderLand_go_temp'
        // 从 Jenkins 凭据获取飞书配置
        FEISHU_APP_ID = credentials('feishu-app-id')
        FEISHU_APP_SECRET = credentials('feishu-app-secret')
        // 飞书用户列表使用全局环境变量
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'test', credentialsId: 'c96914f4-5b65-4230-a456-b9b68f46fcab', url: 'git@github.com:kqq-ai/wonderLand-go.git'
            }
        }

        stage('Build') {
            steps {
                sh '''
                    echo "building..."

                    # 设置国内 Go 代理
                    go env -w GOPROXY=https://goproxy.cn,direct

                    # 更新依赖
                    echo "更新依赖..."
                    go mod tidy

                    # 编译
                    echo "开始编译..."
                    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o cmd/wonderLand ./internal/app/main.go

                    echo "build done!"
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    if [ ! -f "cmd/wonderLand" ]; then
                        echo "错误：编译后的文件不存在"
                        exit 1
                    fi

                    if [ ! -f "build/stop.sh" ]; then
                        echo "错误：stop.sh 脚本文件不存在"
                        exit 1
                    fi

                    echo "准备部署文件..."
                    mkdir -p deploy_temp
                    cp cmd/wonderLand deploy_temp/
                    cp build/stop.sh deploy_temp/
                    
                    echo "打包部署文件..."
                    cd deploy_temp
                    tar -czf ../deploy.tar.gz *
                    cd ..
                    
                    echo "创建远程目录..."
                    ssh -i /root/.ssh/kqq-online.pem -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} "mkdir -p ${REMOTE_DIR} && mkdir -p ${TEMP_DIR}"

                    echo "上传部署包到临时目录..."
                    scp -i /root/.ssh/kqq-online.pem -o StrictHostKeyChecking=no deploy.tar.gz ${REMOTE_USER}@${REMOTE_HOST}:${TEMP_DIR}/

                    echo "停止服务..."
                    ssh -i /root/.ssh/kqq-online.pem -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} "bash -c '\
                        cd ${TEMP_DIR} && \
                        tar -xzf deploy.tar.gz && \
                        cd ${REMOTE_DIR} && \
                        cp -f ${TEMP_DIR}/* ${REMOTE_DIR}/ && \
                        rm -rf ${TEMP_DIR} && \
                        chmod +x wonderLand stop.sh restart.sh && \
                        echo \"开始重启服务...\" && \
                        sh restart.sh'" > deploy.log 2>&1 || true

                    echo "清理本地临时文件..."
                    rm -rf deploy_temp deploy.tar.gz

                    echo "部署完成！"
                '''
            }
        }
    }

    post {
        always {
            script {
                try {
                    // 1. 获取访问令牌
                    echo "开始获取访问令牌..."
                    def tokenRequestBody = """{
                        "app_id": "${FEISHU_APP_ID}",
                        "app_secret": "${FEISHU_APP_SECRET}"
                    }"""
                    echo "Token请求体: ${tokenRequestBody}"
                    
                    def tokenResponse = httpRequest(
                        url: 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal',
                        httpMode: 'POST',
                        contentType: 'APPLICATION_JSON',
                        requestBody: tokenRequestBody
                    )
                    
                    def tokenJson = readJSON text: tokenResponse.content
                    def accessToken = tokenJson.tenant_access_token
                    
                    if (accessToken == null || accessToken.trim() == '') {
                        error "获取访问令牌失败: 令牌为空"
                    }
                    
                    echo "获取到的token: ${accessToken}"

                    // 2. 准备消息内容
                    def buildStatus = currentBuild.result ?: 'SUCCESS'
                    def status = buildStatus == 'SUCCESS' ? '✅ 构建成功' : '❌ 构建失败'
                    
                    // 获取当前时间
                    def currentTime = new Date().format("yyyy-MM-dd HH:mm:ss")
                    
                    // 构建URL
                    def buildUrl = env.BUILD_URL ?: "${env.JENKINS_URL}job/${env.JOB_NAME}/${env.BUILD_NUMBER}"
                    
                    // 计算构建耗时
                    def duration = currentBuild.duration
                    def durationStr = ""
                    if (duration < 1000) {
                        durationStr = "${duration}毫秒"
                    } else if (duration < 60000) {
                        def seconds = (duration/1000).intValue()
                        durationStr = "${seconds}秒"
                    } else {
                        def minutes = (duration/60000).intValue()
                        def seconds = ((duration%60000)/1000).intValue()
                        durationStr = "${minutes}分${seconds}秒"
                    }
                    
                    // 使用\\n来确保换行符被正确转义
                    def messageText = "${status}\\\\n" +
                                    "测试流水线：${JOB_NAME}\\\\n" +
                                    "构建编号：#${BUILD_NUMBER}\\\\n" +
                                    "构建耗时：${durationStr}\\\\n" +
                                    "完成时间：${currentTime}\\\\n" +
                                    "构建地址：${buildUrl}\\\\n" +
                                    "构建结果：${buildStatus}"
                    
                    echo "发送的消息内容: ${messageText}"
                    
                    // 3. 获取用户的 open_id
                    def userList = env.FEISHU_USER_LIST.split(',')
                    def mobileQueryString = userList.collect { "mobiles=${it}" }.join('&')
                    def userResponse = httpRequest(
                        url: "https://open.feishu.cn/open-apis/user/v1/batch_get_id?${mobileQueryString}",
                        httpMode: 'GET',
                        customHeaders: [[
                            name: 'Authorization',
                            value: "Bearer ${accessToken}"
                        ]],
                        validResponseCodes: '100:599'
                    )
                    
                    def userJson = readJSON text: userResponse.content
                    def mobileUsers = userJson.data.mobile_users
                    def mobilesNotExist = userJson.data.mobiles_not_exist ?: []
                    
                    if (mobilesNotExist.size() > 0) {
                        def notExistList = ""
                        mobilesNotExist.each { mobile ->
                            if (notExistList) {
                                notExistList += ", "
                            }
                            notExistList += mobile
                        }
                        echo "警告：以下手机号未找到对应用户：${notExistList}"
                    }
                    
                    // 4. 发送消息给每个有效用户
                    def failedUsers = []
                    
                    mobileUsers.each { mobile, users ->
                        users.each { user ->
                            def userId = user.open_id
                            def messagePayload = """{
                                "receive_id": "${userId}",
                                "msg_type": "text",
                                "content": "{\\\"text\\\":\\\"${messageText}\\\"}"
                            }"""
                            
                            try {
                                def response = httpRequest(
                                    url: 'https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=open_id',
                                    httpMode: 'POST',
                                    customHeaders: [[
                                        name: 'Authorization',
                                        value: "Bearer ${accessToken}"
                                    ], [
                                        name: 'Content-Type',
                                        value: 'application/json; charset=utf-8'
                                    ]],
                                    requestBody: messagePayload.trim(),
                                    validResponseCodes: '100:599'
                                )
                                
                                if (response.status >= 400) {
                                    def errorJson = readJSON text: response.content
                                    echo "发送给用户 ${mobile}(${userId}) 失败: ${errorJson.msg}"
                                    failedUsers.add(mobile)
                                }
                            } catch (Exception e) {
                                echo "发送给用户 ${mobile}(${userId}) 时发生错误: ${e.getMessage()}"
                                failedUsers.add(mobile)
                            }
                        }
                    }
                    
                    // 5. 输出发送结果统计
                    def totalUsers = userList.size()
                    def successUsers = totalUsers - failedUsers.size() - mobilesNotExist.size()
                    echo "消息发送完成，成功：${successUsers}，失败：${failedUsers.size()}，未找到用户：${mobilesNotExist.size()}"
                    if (failedUsers.size() > 0) {
                        def failedList = ""
                        failedUsers.each { mobile ->
                            if (failedList) {
                                failedList += ", "
                            }
                            failedList += mobile
                        }
                        echo "发送失败的用户: ${failedList}"
                    }
                    
                } catch (Exception e) {
                    echo "发送飞书通知失败: ${e.getMessage()}"
                    echo "详细错误: ${e.toString()}"
                    error "发送飞书通知失败"
                }
            }
        }
    }
}