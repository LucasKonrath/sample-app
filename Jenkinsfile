pipeline {
  agent none

  parameters {
    string(name: 'APP_NAME', defaultValue: 'sample-app', description: 'Application / Helm release name')
  }

  environment {
    KUBE_NAMESPACE = 'apps'
    CHART_PATH     = 'charts/app'
    REGISTRY_HOST  = 'registry.infra.svc.cluster.local:5000'
  }

  stages {
    stage('Checkout') {
      agent { label 'built-in' }
      steps { checkout scm }
    }

    stage('Build & Test') {
      agent { label 'built-in' }
      steps { sh 'mvn -pl sample-app -am -B -Dmaven.test.failure.ignore=false test' }
      post { always { junit '**/target/surefire-reports/*.xml' } }
    }

    stage('Build Image & Push (Kaniko)') {
      agent {
        kubernetes {
          yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: maven
      image: maven:3.9-eclipse-temurin-21
      command: ['cat']
      tty: true
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      command: ['cat']
      tty: true
  restartPolicy: Never
"""
        }
      }
      steps {
        container('maven') {
          sh 'mvn -pl sample-app -am -B -DskipTests package'
          sh 'cp sample-app/target/*SNAPSHOT.jar app.jar'
          writeFile file: 'Dockerfile', text: '''
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY app.jar app.jar
EXPOSE 8080 8081
ENTRYPOINT ["java","-jar","/app/app.jar"]
'''
        }
        container('kaniko') {
          sh '''
/kaniko/executor \
  --context=/workspace \
  --dockerfile=/workspace/Dockerfile \
  --destination=${REGISTRY_HOST}/${APP_NAME}:latest \
  --insecure --insecure-pull
'''
        }
      }
    }

    stage('Deploy (Helm)') {
      agent { label 'built-in' }
      steps {
        sh 'helm upgrade --install ' + params.APP_NAME + ' ' + CHART_PATH + ' -n ' + KUBE_NAMESPACE + ' --create-namespace ' + \
           '--set app.name=' + params.APP_NAME + ' --set image.repository=' + REGISTRY_HOST + '/' + params.APP_NAME + ' --set image.tag=latest --set image.pullPolicy=IfNotPresent'
      }
    }
  }

  post {
    success { echo 'Deployment successful.' }
    failure { echo 'Pipeline failed.' }
    always  { echo "Result: ${currentBuild.currentResult}" }
  }
}
