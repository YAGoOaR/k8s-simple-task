#!groovy

pipeline {

    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  name: image-builder
  labels:
    robot: builder
spec:
  serviceAccount: jenkins-agent
  containers:
  - name: jnlp
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.9.1-debug
    imagePullPolicy: Always
    command:
    - /busybox/cat
    tty: true
    volumeMounts:
      - name: docker-config
        mountPath: /kaniko/.docker/
        readOnly: true
  - name: kubectl
    image: bitnami/kubectl
    tty: true
    command:
    - cat
    securityContext:
      runAsUser: 1000
  - name: node
    image: node:16-alpine
    tty: true
    command:
    - cat
  volumes:
    - name: docker-config
      secret:
        secretName: credentials
        optional: false
"""
        }
    }

    parameters {
        string(name: 'IMAGE_NAME', defaultValue: 'yago0ar/express-fe', description: 'Your image name in format USER_NAME/IMAGE. You can write it as default value if you want')
    }

    stages {
        stage('Test') {
            steps {
                // Run 'npm test' using the node container
                // Make sure you do it inside express-fe folder.
                container(name: 'node') {
                    dir (path: './express-fe') {
                        sh 'npm test'
                    }
                }
            }
        }
        stage('Build image') {
            // No need to change, it will work if your Dockerfile is good.
            environment {
                PATH = "/busybox:/kaniko:$PATH"
                HUB_IMAGE = "${params.IMAGE_NAME}"
            }
            steps {
                container(name: 'kaniko', shell: '/busybox/sh') {
                    sh '''#!/busybox/sh
                    /kaniko/executor --dockerfile="$(pwd)/express-fe/Dockerfile" --context="dir:///$(pwd)/express-fe/" --destination ${HUB_IMAGE}:${BUILD_NUMBER}
                    '''
                }
            }
        }
        stage('Deploy') {
            steps {
                // Need to do two things
                // First: somehow using bash, substitute new params.IMAGE_NAME and BUILD_NUMBER variable into your frontend deployment.
                // Hint: bitnami/kubectl has 'sed' utility available
                // But you can use any other solution (Kustomize, etc.)
                // Second - use kubectl apply from kubectl container

                container(name: 'kubectl') {
                    dir (path: './k8s/') {
                        sh """cat <<EOF >./kustomization.yaml
resources:
- backend-deployment.yaml
- frontend-deployment.yaml
- backend-service.yaml
- frontend-service.yaml
- mysql-service.yaml
- mysql-stateful.yaml
images:
- name: yago0ar/express-fe
  newName: "${params.IMAGE_NAME}"
  newTag: "${BUILD_NUMBER}" 
EOF"""
                        sh 'kubectl apply --kustomize ./'
                    }
                }
            }
        }
        stage('Test deployment') {
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
metadata:
  name: tester
  labels:
    robot: tester
spec:
  serviceAccount: jenkins-agent
  containers:
  - name: jnlp
  - name: ubuntu
    image: ubuntu:22.04
    tty: true
    command:
    - cat
"""
                }
            }
            steps {
                // Using ubuntu container install `curl`
                // Use curl to make a request to curl http://frontend:80/books
                // You probably have to wait for like 60-120 second till everything is deployed for the first time
                container(name: 'ubuntu') {
                    sh 'apt-get update --yes'
                    sh 'apt-get install curl --yes'
                    sh 'sleep 100'
                    sh 'curl http://frontend:80/books'
                }
            }
        }
    }
}
