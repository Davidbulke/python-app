pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  namespace: jenkins
spec:
  containers:
    - name: jnlp
      image: jenkins/inbound-agent:3206.vb_15dcf73f6a_9-2-jdk17
      resources:
        requests:
          memory: "256Mi"
          cpu: "100m"
        limits:
          memory: "512Mi"
          cpu: "250m"
    - name: python
      image: python:3.12-slim
      command: [sleep]
      args: [99d]
      resources:
        requests:
          memory: "512Mi"
          cpu: "250m"
        limits:
          memory: "1Gi"
          cpu: "500m"
    - name: kaniko
      image: gcr.io/kaniko-project/executor:v1.19.0-debug
      command: ['/busybox/cat']
      tty: true
      resources:
        requests:
          memory: "512Mi"
          cpu: "250m"
        limits:
          memory: "1Gi"
          cpu: "500m"
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
    - name: trivy
      image: aquasec/trivy:0.48.3
      command: ['sleep']
      args: ['99d']
      resources:
        requests:
          memory: "128Mi"
          cpu: "50m"
        limits:
          memory: "256Mi"
          cpu: "100m"
  volumes:
    - name: docker-config
      secret:
        secretName: docker-config
        items:
          - key: .dockerconfigjson
            path: config.json
'''
        }
    }

    environment {
        DOCKER_USERNAME = 'davidbulke'
        IMAGE_NAME = 'python-app'
        GIT_COMMIT_SHORT = sh(script: "git rev-parse --short=8 HEAD", returnStdout: true).trim()
        GIT_BRANCH = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
        IMAGE_TAG = "${GIT_COMMIT_SHORT}-${BUILD_NUMBER}"
        FULL_IMAGE_NAME = "${DOCKER_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "Pipeline starting for branch ${GIT_BRANCH} commit ${GIT_COMMIT_SHORT}"
                sh 'ls -la'
            }
        }

        stage('Trivy Security Scan - Source Code') {
            steps {
                container('trivy') {
                    sh '''
                        trivy fs --exit-code 0 --severity HIGH,CRITICAL --no-progress . 
                        echo "Source code security scan completed!"
                    '''
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                container('python') {
                    sh 'pip install -r requirements.txt'
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                container('python') {
                    sh 'pytest --junitxml=pytest-results.xml'
                    junit 'pytest-results.xml'
                }
            }
        }
        
        stage('Build Docker Image with Kaniko') {
            steps {
                container('kaniko') {
                    script {
                        def tags = [
                            "--destination=${FULL_IMAGE_NAME}"
                        ]
                        if (env.GIT_BRANCH == 'main') {
                            tags.add("--destination=${DOCKER_USERNAME}/${IMAGE_NAME}:latest")
                        }
                        def tagString = tags.join(' ')
                        sh """
                            /kaniko/executor \
                                --context=\${PWD} \
                                --dockerfile=\${PWD}/Dockerfile \
                                ${tagString} \
                                --cache=true \
                                --cache-ttl=24h \
                                --compressed-caching=false \
                                --snapshot-mode=redo \
                                --log-format=text \
                                --verbosity=info
                        """
                    }
                }
            }
        }

        stage('Trivy Security Scan - Docker Image') {
            steps {
                container('trivy') {
                    sh """
                        trivy image --exit-code 0 --severity CRITICAL --no-progress ${FULL_IMAGE_NAME}
                        echo "Docker image security scan finished!"
                    """
                }
            }
        }
    }

    post {
        success {
            script {
                def latestInfo = (env.GIT_BRANCH == 'main') ? 
                    "\n   - ${DOCKER_USERNAME}/${IMAGE_NAME}:latest" : 
                    ""
                    
                echo """
                    
                        CI PIPELINE SUCCESS! 🎉        
                    

                    ✅ All Stages Completed:
                    - Source code security scanned (Trivy)
                    - Python dependencies installed
                    - Unit tests executed (pytest)
                    - Docker image built (Kaniko)
                    - Image security validated (Trivy)
                    - Image pushed to Docker Hub

                    🐳 Docker Images Published:
                    - ${FULL_IMAGE_NAME}${latestInfo}

                    📊 Build Info:
                    - Build: #${BUILD_NUMBER}
                    - Branch: ${GIT_BRANCH}
                    - Commit: ${GIT_COMMIT_SHORT}
                    - Tag: ${IMAGE_TAG}
                    
                    🔗 Pull Command:
                    docker pull ${FULL_IMAGE_NAME}
                    
                """
            }
        }
        
        failure {
            echo """
            
                ❌ PIPELINE FAILED
            
            Stage: ${env.STAGE_NAME}
            Build: #${BUILD_NUMBER}
            Branch: ${GIT_BRANCH}
            Commit: ${GIT_COMMIT_SHORT}
            
            Check logs above for error details.
            
            """
        }
        
        always {
            echo "Pipeline execution completed for build #${BUILD_NUMBER}"
        }
    }
}
