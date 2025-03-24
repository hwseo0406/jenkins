pipeline {
  agent {
    kubernetes {
      defaultContainer 'kaniko'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: ["sleep"]
    args: ["3600"]
    volumeMounts:
    - name: harbor-creds
      mountPath: /kaniko/.docker
    - name: workspace-volume
      mountPath: /home/jenkins/agent
  - name: jnlp
    image: jenkins/inbound-agent:latest
  volumes:
  - name: harbor-creds
    secret:
      secretName: harbor-creds
      items:
      - key: .dockerconfigjson
        path: config.json
  - name: workspace-volume
    emptyDir: {}
"""
    }
  }

  environment {
    IMAGE_NAME = "harbor.locall:30004/jentest/nginx"
    IMAGE_TAG = "${BUILD_NUMBER}"
    FULL_IMAGE = "${IMAGE_NAME}:${IMAGE_TAG}"
  }

  stages {
    stage('Checkout App Repo') {
      steps {
        git credentialsId: 'github-token', url: 'https://github.com/hwseo0406/jenkins.git', branch: 'main'
      }
    }

    stage('Build & Push with Kaniko') {
      steps {
        container('kaniko') {
          sh """
            echo '📁 현재 위치: ' `pwd`
            echo '📄 index.html 파일 있는지 확인:'
            ls -al
            cat index.html || echo '❌ index.html 없음'

            /kaniko/executor \
              --context `pwd` \
              --dockerfile `pwd`/Dockerfile \
              --destination=${FULL_IMAGE} \
              --insecure \
              --skip-tls-verify

            /kaniko/executor \
              --context `pwd` \
              --dockerfile `pwd`/Dockerfile \
              --destination=${IMAGE_NAME}:latest \
              --insecure \
              --skip-tls-verify
          """
        }
      }
    }

    stage('Clone Manifests Repo') {
      steps {
        container('jnlp') {
          withCredentials([usernamePassword(credentialsId: 'hwseo github token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/hwseo0406/manifests.git
            """
          }
        }
      }
    }

    stage('Update Manifest & Push') {
      steps {
        container('jnlp') {
          withCredentials([usernamePassword(credentialsId: 'hwseo github token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            script {
              def newImage = "${FULL_IMAGE}"
              sh """
                sed -i 's|image: .*|image: ${newImage}|' manifests/jentest-deployment.yaml
                cat manifests/jentest-deployment.yaml

                cd manifests
                git config --global user.email "hse05078@gmail.com"
                git config --global user.name "hwseo0406"
                git add jentest-deployment.yaml
                git commit -m "Update image to ${newImage} [skip ci]" || echo "No changes to commit"
                git push origin main
              """
            }
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('jnlp') {
          withCredentials([string(credentialsId: 'jenkinsSA', variable: 'K8S_TOKEN')]) {
            sh """
              echo "📦 kubectl 설치 중 (임시)"
              apt-get update && apt-get install -y curl gnupg apt-transport-https ca-certificates
              curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
              echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
              apt-get update && apt-get install -y kubectl

              echo "🔧 kubeconfig 생성"
              cat <<EOF > kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://172.18.0.4:6443
    insecure-skip-tls-verify: true
  name: jenkins-cluster
users:
- name: jenkins
  user:
    token: ${K8S_TOKEN}
contexts:
- context:
    cluster: jenkins-cluster
    user: jenkins
  name: jenkins-context
current-context: jenkins-context
EOF

              export KUBECONFIG=kubeconfig.yaml

              echo "🚀 jentest-deployment에 새 이미지 적용"
              kubectl set image deployment/jentest-deployment nginx=${FULL_IMAGE} -n test

              echo "🔄 롤링 업데이트 상태 확인"
              kubectl rollout status deployment/jentest-deployment -n test

              echo "🧹 kubeconfig 정리"
              rm -f kubeconfig.yaml
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ 이미지 빌드 및 배포 성공: ${FULL_IMAGE}"
    }
    failure {
      echo "❌ 실패. 로그를 확인하세요."
    }
  }
}
