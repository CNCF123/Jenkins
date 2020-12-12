pipeline {

  agent {
    kubernetes {
      cloud 'kubernetes-default'
      slaveConnectTimeout 1200
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jnlp
      image: "jenkins/inbound-agent:4.6-1-alpine"
      args: [\'$(JENKINS_SECRET)\', \'$(JENKINS_NAME)\']
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
        - mountPath: "/etc/hosts"
          name: "volume-hosts"
          readOnly: false        
    - name: "maven-3.6.3-jdk-8"
      command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
      image: "maven:3.6.3-jdk-8"
      imagePullPolicy: "IfNotPresent"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
        - mountPath: "/root/.m2/"
          name: "volume-maven-repo"
          readOnly: false
        - mountPath: "/etc/hosts"
          name: "volume-hosts"
          readOnly: false
    - name: "maven-3.6.3-jdk-11"
      command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
      image: "maven:3.6.3-jdk-11"
      imagePullPolicy: "IfNotPresent"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
        - mountPath: "/root/.m2/"
          name: "volume-maven-repo"
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
      image: "syseleven/kubectl-helm:helm-3.2.4" ###该镜像需要制作，还需要kubeconf信息
      imagePullPolicy: "IfNotPresent"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "volume-time"
          readOnly: false
        - mountPath: "/var/run/docker.sock"
          name: "volume-docker"
          readOnly: false
        - mountPath: "/mnt/.kube/"
          name: "volume-kubeconfig"
          readOnly: false
        - mountPath: "/etc/hosts"
          name: "volume-hosts"
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
      image: "docker:19.03.9-git"
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
  restartPolicy: "Never"
  nodeSelector:
    build: "true"
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
    - name: "volume-maven-repo"
      hostPath:
        path: "/opt/m2"
    - name: "volume-kubeconfig"
      secret:
        secretName: "multi-kube-config"
''' 
}
}

  stages {
    stage('pulling Code') {
      parallel {
        stage('pulling Code') {
          when {
            expression {
              env.gitlabBranch == null
            }
          }
          steps {
            git(branch: "${BRANCH}", credentialsId: 'cdce3d8e-a859-45ac-9926-ac34236bb744', url: "${REPO_URL}")
          }
        }

        stage('pulling Code by trigger') {
          when {
            expression {
              env.gitlabBranch != null
            }
          }
          steps {
            git(url: "${REPO_URL}", branch: env.gitlabBranch, credentialsId: 'cdce3d8e-a859-45ac-9926-ac34236bb744')
          }
        }

      }
    }

    stage('initConfiguration') {
      steps {
        script {
          CommitID = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
          CommitMessage = sh(returnStdout: true, script: "git log -1 --pretty=format:'%h : %an  %s'").trim()
          def curDate = sh(script: "date '+%Y%m%d-%H%M%S'", returnStdout: true).trim()
          TAG = curDate[0..14] + "-" + CommitID + "-" + BRANCH
        }

      }
    }

    stage('Building') {
      parallel {
        stage('Building') {
          steps {
            container(name: 'maven-3.6.3-jdk-8') {
            sh """
            echo "Building Project..."
            ${BUILD_COMMAND}
          """
            }

          }
        }

        stage('Scan Code') {
          steps {
            sh 'echo "Scan Code"'
          }
        }

      }
    }

    stage('Build image') {
      steps {
                withCredentials([usernamePassword(credentialsId: 'REGISTRY_USER', passwordVariable: 'Password', usernameVariable: 'Username')]) {
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

    stage('Deploy') {
    when {
            expression {
              DEPLOY != "false"
            }
          }
    
      steps {
      container(name: 'kubectl-helm') {
        sh """
        cat ${KUBECONFIG_PATH} > /tmp/1.yaml
  /usr/local/bin/kubectl config use-context ${CLUSTER} --kubeconfig=/tmp/1.yaml
  export KUBECONFIG=/tmp/1.yaml
  /usr/local/bin/kubectl set image ${DEPLOY_TYPE} -l ${DEPLOY_LABEL} ${CONTAINER_NAME}=${HARBOR_ADDRESS}/${REGISTRY_DIR}/${IMAGE_NAME}:${TAG} -n ${NAMESPACE}

  应该使用helm部署
"""
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
