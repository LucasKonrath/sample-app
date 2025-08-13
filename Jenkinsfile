pipeline {
  agent any
  options { timestamps() }
  environment {
    APP_NAME = 'sample-app'
    KUBE_NAMESPACE = 'apps'
    CHART_PATH = 'charts/app'
    REGISTRY_HOST = 'registry.infra.svc.cluster.local:5000'
    IMAGE_FULL = "${REGISTRY_HOST}/${APP_NAME}:latest"
  }
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Build & Test') {
      steps { sh 'mvn -B -Dmaven.test.failure.ignore=false test' }
      post { always { junit '**/target/surefire-reports/*.xml' } }
    }
    stage('Build Image (Kaniko)') {
      agent {
        kubernetes {
          inheritFrom ''
          yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins/label: kaniko-build
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      args:
        - --dockerfile=Dockerfile
        - --context=workspace/sample-app
        - --destination=${REGISTRY_HOST}/${APP_NAME}:latest
        - --insecure
        - --insecure-pull
      volumeMounts:
        - name: workspace
          mountPath: /workspace
    - name: maven
      image: maven:3.9-eclipse-temurin-21
      command: ['cat']
      tty: true
      volumeMounts:
        - name: workspace
          mountPath: /workspace
  volumes:
    - name: workspace
      emptyDir: {}
"""
        }
      }
      steps {
        container('maven') {
          sh 'cd sample-app && mvn -B -DskipTests package && cp target/*-SNAPSHOT.jar ../Dockerfile || true'
        }
        script {
          writeFile file: 'Dockerfile', text: """
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY sample-app/target/*SNAPSHOT.jar app.jar
EXPOSE 8080 8081
ENTRYPOINT ["java","-jar","/app/app.jar"]
"""
        }
        container('kaniko') {
          sh 'echo Building with Kaniko'
        }
      }
    }
    stage('Deploy (Helm)') {
      steps {
        sh "helm upgrade --install ${APP_NAME} ${CHART_PATH} -n ${KUBE_NAMESPACE} --create-namespace --set app.name=${APP_NAME} --set image.repository=${REGISTRY_HOST}/${APP_NAME} --set image.tag=latest --set image.pullPolicy=IfNotPresent"
      }
    }
  }
  post { failure { echo 'Pipeline failed.' } success { echo 'Deployment successful.' } }
}
