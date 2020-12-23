pipeline {
  agent {
    kubernetes {
      cloud "${kubernetes_name}"
      slaveConnectTimeout 1200
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jnlp
      args: [\'$(JENKINS_SECRET)\', \'$(JENKINS_NAME)\']
      image: 'registry.cn-hangzhou.aliyuncs.com/pipeline-cicd/jenkins-inbound-agent:4.3-9-alpine'
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
          
    - name: "build"
      command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
      image: "registry.cn-hangzhou.aliyuncs.com/pipeline-cicd/maven:3.5.3"
      imagePullPolicy: "IfNotPresent"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
        - mountPath: "/root/.m2/repository"
          name: "volume-maven-repo"
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
      image: "registry.cn-hangzhou.aliyuncs.com/pipeline-cicd/docker:19.03"
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
          
    - name: "kubectl"
      command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
      image: "registry.cn-beijing.aliyuncs.com/citools/kubectl:1.17.4"
      imagePullPolicy: "IfNotPresent"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
        - mountPath: "/etc/hosts"
          name: "volume-hosts"
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
    stage('pulling code') {
      steps {
        git(url: "${GIT_URL}", branch: "${BRANCH}", changelog: true, credentialsId: 'gitlab-auth')
      }
    }

    stage('initConfig') {
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
        container(name: 'build') {
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
        configFileProvider([configFile(fileId: "d3b40aee-6426-414c-aac0-af9f3df56b29", targetLocation: "admin.kubeconfig")]){
          container(name: 'kubectl') {  
            sh """
            kubectl set image ${DEPLOY_TYPE} -l ${DEPLOY_LABEL} ${CONTAINER_NAME}=${HARBOR_ADDRESS}/${REGISTRY_DIR}/${IMAGE_NAME}:${TAG} -n ${NAMESPACE} --kubeconfig=admin.kubeconfig
            """
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
