pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
    - name: maven
      image: maven:3.9-eclipse-temurin-21
      command: ['cat']
      tty: true
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      command: ['cat']
      tty: true
    - name: helm
      image: alpine/helm:3.14.0
      command: ['cat']
      tty: true
"""
    }
  }

  parameters {
    string(name: 'APP_NAME', defaultValue: 'sample-app', description: 'App / Helm release name')
  }

  environment {
    KUBE_NAMESPACE = 'apps'
    CHART_PATH     = 'charts/app'
    REGISTRY_HOST  = 'registry.infra.svc.cluster.local:5000'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Test') {
      steps {
        container('maven') {
          sh 'mvn -pl sample-app -am -B test'
        }
      }
    }

    stage('Package & Dockerfile') {
      steps {
        container('maven') {
          sh 'mvn -pl sample-app -am -B -DskipTests package'
          writeFile file: 'Dockerfile', text: '''
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY sample-app/target/*SNAPSHOT.jar app.jar
EXPOSE 8080 8081
ENTRYPOINT ["java","-jar","/app/app.jar"]
'''
        }
      }
    }

    stage('Build Image (Kaniko)') {
      steps {
        container('kaniko') {
          sh '''
/kaniko/executor \
  --context=${PWD} \
  --dockerfile=${PWD}/Dockerfile \
  --destination=${REGISTRY_HOST}/${APP_NAME}:latest \
  --insecure --insecure-pull
'''
        }
      }
    }

    stage('Deploy (Helm)') {
      steps {
        container('helm') {
          sh 'helm upgrade --install ' + params.APP_NAME + ' ' + CHART_PATH + ' -n ' + KUBE_NAMESPACE + ' --create-namespace ' + \
             '--set app.name=' + params.APP_NAME + ' --set image.repository=' + REGISTRY_HOST + '/' + params.APP_NAME + ' --set image.tag=latest --set image.pullPolicy=IfNotPresent'
        }
      }
    }
  }

  post {
    success { echo 'Deployment successful.' }
    failure { echo 'Pipeline failed.' }
    always  { echo "Result: ${currentBuild.currentResult}" }
  }
}
