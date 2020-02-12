import hudson.model.*;
 
println env.JOB_NAME
println env.BUILD_NUMBER
println env.WORKSPACE
println env.BRANCH_NAME

pipeline {

    options { disableConcurrentBuilds() }

    environment {
		GIT_TAG = sh(returnStdout: true,script: 'git describe --tags --always').trim()
        PROJECT_NAME = "go-ocr" 
		ENV_NAME = "${env.BRANCH_NAME == 'master' ? 'pro' : 'dev'}"
        TAG_NAME = "${env.BRANCH_NAME == 'master' ? 'stable' : 'test'}.${env.BUILD_NUMBER}.${GIT_TAG}"
        //TAG_NAME = 'latest'
		REGISTRY_CREDENTIALS_ID = 'jenkins-registry-admin'
		ESCLOUD_TOKEN='kuboard-escloud-token'
		DOCKER_ADDR = 'http://registry.apps.es1688.k8s.local'
		def MAVEN_HOME  = tool 'mvn3.6'
		def NODE_HOME = tool 'nodejs13'

		//env.PATH = "${MAVEN_HOME}/bin:${env.PATH}"
    }
	//在master节点配置jenkins的k8s云参数
    agent { label "master" }

    stages {
		//复制jenkins配置文件
        stage('准备配置文件') {
            steps {
                sh "ls -lrt && pwd"

				sh "cp -rf ${JENKINS_HOME}/mvnconf/* ${MAVEN_HOME}/conf/"
				
                //sh "mkdir -p ${JENKINS_HOME}/jenkins_config/"
                //sh "cp -rf * ${JENKINS_HOME}/jenkins_config/"
            }
        }
		//maven编译项目并打包
        stage('服务端编译') {
			steps {
				//dir('./common') {
				//	sh "${MAVEN_HOME}/bin/mvn  install -Dmaven.test.skip=true"
				//}
			   sh "${MAVEN_HOME}/bin/mvn install -Dmaven.test.skip=true"
			}
            //steps { sh '${JENKINS_HOME}/consul-template -vault-addr "http://consul.es-jr.cn" -config "jenkins_config.hcl" -once -vault-retry-attempts=1 -vault-renew-token=false' }
        }

		//在master节点上执行脚本设置jenkins配置k8s云参数，主要是jenkins-slave参数
        stage('设置云参数') {
            steps {
				  sh "ls -lrt && pwd"
                //load("/var/jenkins_home/jenkins_config/src/kubernetes.groovy")
            }
        }
		//通过dockerfile创建Docker文件
        stage('构建服务端镜像') {
            steps {
                script {
			        def escloud_data =''
			        def IMAGE_NAME = ''
				    def CURL_CMD=''
                    def CURL_RESULT=''
                    //TODO：prolist需要修改成对应的项目模块和路径
					def prolist = ['go-ocr:/']
					//TODO：SERVER_NAME需要修改成对应的Kuboard的pod值
			        def SERVER_NAME = "svc-go-ocr"
			        def KUBOARD_URL = "http://10.13.32.94:32567/k8s-api/apis/apps/v1/namespaces/es-cloud/deployments/${SERVER_NAME}"
                    def WEIXIN_URL = 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key='
                    def WEIXIN_DATA = ''
			        
					for (int i = 0;i < prolist.size();++i){
						def prolistitem = prolist[i].tokenize(':')
						def proj_path = prolistitem[1]
						def proj_name = prolistitem[0]
						echo "docker build the ${prolist[i]} project"
						echo "项目名称 ${proj_name} ,路径:${proj_path} "
						docker.withRegistry('http://registry.apps.es1688.k8s.local', "${REGISTRY_CREDENTIALS_ID}") {
							def CUR_Docker = docker.build("${PROJECT_NAME}-${ENV_NAME}/${proj_name}:${TAG_NAME}", "-f ${proj_path}/Dockerfile ${proj_path}")
							CUR_Docker.push()
							CUR_Docker.push('latest')							//发布应用
							IMAGE_NAME = "registry.apps.es1688.k8s.local/${PROJECT_NAME}-${ENV_NAME}/${proj_name}:${TAG_NAME}"
				            //escloud_data='{"spec":{"template":{"spec":{"containers":[{"name":"'+ proj_name +'","image":"'+IMAGE_NAME +'"}]}}}}'
                            escloud_data='{\\"spec\\":{\\"template\\":{\\"spec\\":{\\"containers\\":[{\\"name\\":\\"'+proj_name+'\\",\\"image\\":\\"'+IMAGE_NAME+'\\"}]}}}}'

							echo "${escloud_data}"
							withCredentials([string(credentialsId: 'kuboard-escloud-token', variable: 'ESCLOUD_TOKEN_TEXT')]) {
								CURL_CMD='curl -X PATCH  -H "content-type: application/strategic-merge-patch+json" -H "'+ESCLOUD_TOKEN_TEXT+'"  -d  "'+escloud_data+'" "'+KUBOARD_URL+'"'
                               // CURL_RESULT = sh label: 'deploy', returnStdout: true, script: CURL_CMD
                                echo "CURL_RESULT：${CURL_RESULT}"
							 }
            				withCredentials([string(credentialsId: 'jenkins-weixin-robot', variable: 'WEIXIN_KEY')]) {
            				    WEIXIN_DATA = '{\\"msgtype\\": \\"markdown\\",\\"markdown\\":{\\"content\\": \\"项目<font color=\\\\"#FF00FF\\\\">'+proj_name+'</font>部署成功。\\n >Jenkins任务:<font color=\\\\"#FF00FF\\\\">'+PROJECT_NAME+'</font> \\n >环境:<font color=\\\\"#FF00FF\\\\">'+ENV_NAME+'</font> \\n >Docker镜像:<font color=\\\\"#FF00FF\\\\">'+IMAGE_NAME+'</font>\\" }}'
                                WEIXIN_URL = WEIXIN_URL + WEIXIN_KEY
            				    CURL_CMD='curl   -H "Content-Type: application/json"   -d  "'+WEIXIN_DATA+'" "'+WEIXIN_URL+'"'
                                echo "CURL_CMD：${CURL_CMD}"
                               // CURL_RESULT = sh label: 'weixin', returnStdout: true, script: CURL_CMD
                                echo "CURL_RESULT：${CURL_RESULT}"
            				}

						}						
					}
                }
            }
        }
		//通过dockerfile创建Docker文件
        stage('监控服务启动状态') {
            steps {
                script {
					 def proj_name = 'escloud-admin'
					 //def proj_path = 'escloud-admin'
					 
					

                }
            }
        }

    }
}
