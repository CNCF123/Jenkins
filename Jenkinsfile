pipeline {
  agent {
    kubernetes {
      cloud "${K8S_NAME}"
      slaveConnectTimeout 1200
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jnlp
      args: [\'$(JENKINS_SECRET)\', \'$(JENKINS_NAME)\']
      image: 'registry.cn-hangzhou.aliyuncs.com/cicd-liu-shui-xian/jenkins-inbound-agent:4.10-3-alpine'
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
    - name: "maven38jdk8"
      command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
      image: "registry.cn-hangzhou.aliyuncs.com/cicd-liu-shui-xian/maven:3.8-jdk8"
      imagePullPolicy: "IfNotPresent"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
        - mountPath: "/root/.m2/repository"
          name: "volume-maven-repo"
          readOnly: false
    - name: "maven38jdk11"
      command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
      image: "registry.cn-hangzhou.aliyuncs.com/cicd-liu-shui-xian/maven:3.8-jdk11"
      imagePullPolicy: "IfNotPresent"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
        - mountPath: "/root/.m2/repository"
          name: "volume-maven-repo"
          readOnly: false
    - name: "golang"
      command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
      image: "registry.cn-hangzhou.aliyuncs.com/cicd-liu-shui-xian/golang:1.18"
      imagePullPolicy: "IfNotPresent"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false    
    - name: "docker"
      command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
      image: "docker:19.03"
      imagePullPolicy: "IfNotPresent"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
        - mountPath: "/var/run/docker.sock"
          name: "volume-docker"
          readOnly: false
        - mountPath: "/etc/hosts"
          name: "volume-hosts"
          readOnly: false        
    - name: "kubectl-helm"
      command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
      image: "registry.cn-hangzhou.aliyuncs.com/pipeline-cicd/kubectl-helm:1.18.8-3.3.3"
      imagePullPolicy: "IfNotPresent"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
  restartPolicy: "Never"
  securityContext: {}
  volumes:
    - hostPath:
        path: "/var/run/docker.sock"
      name: "volume-docker"
    - hostPath:
        path: "/usr/share/zoneinfo/Asia/Shanghai"
      name: "volume-time"
    - hostPath:
        path: "/etc/hosts"
      name: "volume-hosts"
    - hostPath:
        path: "/tmp/m2"
      name: "volume-maven-repo"
''' 
}
}

  stages {
      
    stage('pull code') {
      steps {
        git(url: "${GIT_URL}", branch: "${BRANCH}", changelog: true, credentialsId: 'gitlab-auth')
      }
    }

    stage('initTag') {
      steps {
        script {
          CommitID = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
          CommitMessage = sh(returnStdout: true, script: "git log -1 --pretty=format:'%h : %an  %s'").trim()
          def curDate = sh(script: "date '+%Y%m%d-%H%M%S'", returnStdout: true).trim()
          TAG = curDate[0..14] + "-" + CommitID + "-" + BRANCH
        }
      }
    }

    stage('build') {
      steps {
        container(name: "${BUILD_TYPE}") {
          sh """
          echo "Building Project..."
          ${BUILD_COMMAND}
          """
        }
      }
    }

    stage('build docker image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-auth', passwordVariable: 'Password', usernameVariable: 'Username')]) {
          container(name: 'docker') {
            sh """
            docker build -t ${HARBOR_ADDRESS}/${REGISTRY_DIR}/${IMAGE_NAME}:${TAG} .
            docker login -u ${Username} -p ${Password} ${HARBOR_ADDRESS}
            docker push ${HARBOR_ADDRESS}/${REGISTRY_DIR}/${IMAGE_NAME}:${TAG}
            """
          }
        }
      }
    }

    stage('deploy') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-auth', passwordVariable: 'Password', usernameVariable: 'Username')]) {
          configFileProvider([configFile(fileId: "${KUBECONFIG_NAME}", targetLocation: "admin.kubeconfig")]){
            container(name: 'kubectl-helm') {  
            sh """
            kubectl create namespace ${NAMESPACE} --kubeconfig=admin.kubeconfig | true
            kubectl create secret docker-registry image-pull-secret --docker-server=${HARBOR_ADDRESS} --docker-username=${Username} --docker-password=${Password} -n ${NAMESPACE} --kubeconfig=admin.kubeconfig | true
            
            helm repo add harbor-helm-chart --username=${Username} --password=${Password}  https://${HARBOR_ADDRESS}/chartrepo/helm-chart
            if helm history \${APP} -n \${NAMESPACE} --kubeconfig=admin.kubeconfig &> /dev/null
              then
              action=upgrade
            else
              action=install
            fi
            helm_args="--set container.image.repository=${HARBOR_ADDRESS}/${REGISTRY_DIR}/${IMAGE_NAME} --set container.image.tag=${TAG} --set container.port=${PORT}"
            if [ ${COMMAND} != "" ]
            then
              helm_args="${helm_args} --set container.command=${COMMAND}"
            fi
            if [ ${HPA} == true ]
            then
              helm_args="${helm_args} --set hpa.enable=true"
            fi
            helm \${action} ${APP} harbor-helm-chart/deployment -n ${NAMESPACE} \${helm_args} --kubeconfig=admin.kubeconfig
            """
            }
          }
        }  
      }
    }

  }
  environment {
    CommitID = ''
    CommitMessage = ''
    TAG = ''
  }
}
