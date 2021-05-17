#!groovy

import java.text.SimpleDateFormat

node() {
    // 镜像tag用时间戳显示，唯一性
    def dateFormat = new SimpleDateFormat("yyyyMMddHHmm")
    def dockerTag = dateFormat.format(new Date())

    // 个人阿里云镜像仓库地址及命名空间
    def registry = 'registry.cn-beijing.aliyuncs.com'
    def aliyunNamespace = 'marco_images/image-test'

    // 部署项目的服务器ip
    def sshIP = '8.140.26.173'
    def dockerName = 'marco-test'

    stage('get souce code') {
        try {
            echo "get source code"
            checkout scm
        }
        catch(err) {
            echo "get source code failed"
            throw err
        }
    }

    stage('npm run build') {
        try{
            docker.image('node:12-alpine').inside {
                sh "npm --registry https://registry.npm.taobao.org install"
                sh 'npm install'
                sh 'npm run build'
            }
        }
        catch(err) {
            echo "npm run build failed"
            throw err
        }
    }

    stage('build & upload Image') {
        withCredentials([usernamePassword(credentialsId: 'jenkins_login_aliyun_docker', usernameVariable: 'username', passwordVariable: 'password')]) {
            try {
                sh 'cp -r dist ./devops_build'

                sh "docker login -u ${username} -p ${password} ${registry}"

                sh "docker build -t ${registry}/${aliyunNamespace} ./devops_build"

                sh "docker tag ${registry}/${aliyunNamespace} ${registry}/${aliyunNamespace}:${dockerTag}"

                sh "docker push ${registry}/${aliyunNamespace}:${dockerTag}"

                sh "docker rmi ${registry}/${aliyunNamespace}:${dockerTag}"
            }
            catch(err) {
                echo "build and upload Image failed"
                throw err
            }
        }
    }

    stage('ssh server & pull image'){
        try {
            // 连接远程服务器
            def sshServer = getServer(sshIP)
            // 更新或下载镜像
            sshCommand remote: sshServer, command: "docker pull ${registry}/${aliyunNamespace}:${dockerTag}"
            
            // 停止并删除容器
            sshCommand remote: sshServer, command: "docker rm -f ${dockerName}"
            // 启动
            sshCommand remote: sshServer, command: "docker run -u root --name ${dockerName} -p 80:80 -d ${registry}/${aliyunNamespace}:${dockerTag}"
            // 只保留3个最新的镜像
            sshCommand remote: sshServer, command: """docker rmi -f \$(docker images | grep ${registry}/${aliyunNamespace} | sed -n  '4,\$p' | awk '{print \$3}') || true"""
        }
        catch(err){
            echo "remote & pull image failed"
            throw err
        }
    }
}

def getServer(ip){
    def remote = [:]
    remote.name = "server-${ip}"
    remote.host = ip
    remote.port = 22
    remote.allowAnyHosts = true
    withCredentials([usernamePassword(credentialsId: 'ssh_remote_server', passwordVariable: 'password', usernameVariable: 'username')]) {
        remote.user = "${username}"
        remote.password = "${password}"
    }
    return remote
}
