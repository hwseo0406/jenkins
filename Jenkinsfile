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
  - name: kubectl
    image: gcr.io/cloud-builders/kubectl
    command: ["cat"]
    tty: true
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

    stage('Skip CI Check') {
      steps {
        container('jnlp') {
          script {
            def skip = sh(
              script: "git log -1 --pretty=%B | grep -c '\\[skip ci\\]' || true",
              returnStdout: true
            ).trim()
            if (skip == "1") {
              echo "⏭️ [skip ci] 커밋이 감지되어 파이프라인을 종료합니다."
              currentBuild.result = 'SUCCESS'
              error("[skip ci] detected - stopping pipeline.")
            }
          }
        }
      }
    }

    stage('Check Kaniko Config') {
      steps {
        container('kaniko') {
          sh 'cat /kaniko/.docker/config.json || echo "❌ config.json not found"'
        }
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
        container('kubectl') {
          withKubeConfig([
            serverUrl: 'https://172.18.0.4:6443',
            credentialsId: 'jenkinsSA',
          ]) {
            sh """
              kubectl apply -f manifests/jentest-deployment.yaml
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ 이미지 빌드, 매니페스트 업데이트 및 배포 성공: ${FULL_IMAGE}"
    }
    failure {
      echo "❌ 실패. 로그를 확인하세요."
    }
  }
}
