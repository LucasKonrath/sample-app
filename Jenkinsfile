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
      image: gcr.io/kaniko-project/executor:debug
      command: ['sh','-c']
      args: ['sleep 3600']
      tty: true
    - name: helm
      # Switched image to include kubectl
      image: dtzar/helm-kubectl:3.14.0
      command: ['cat']
      tty: true
"""
    }
  }

  parameters {
    string(name: 'APP_NAME', defaultValue: 'sample-app', description: 'App / Helm release name')
    string(name: 'UPSTREAM_BASE_IMAGE', defaultValue: 'docker.io/library/eclipse-temurin:21-jre', description: 'Upstream public base image to mirror')
    string(name: 'MIRRORED_BASE_IMAGE', defaultValue: 'base/eclipse-temurin:21-jre', description: 'Internal mirrored base image path (suffix after registry host)')
    string(name: 'CHART_PATH', defaultValue: 'charts/app', description: 'Relative path to Helm chart within repo (searched if missing)')
    string(name: 'REGISTRY_HOST_OVERRIDE', defaultValue: '', description: 'Optional manual registry host:port override (takes precedence)')
    string(name: 'REGISTRY_NODEPORT', defaultValue: '32000', description: 'NodePort exposing the Minikube addon registry (must match infra)')
  }

  environment {
    KUBE_NAMESPACE = 'apps'
    REGISTRY_HOST = 'unset'
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Resolve Registry Host') {
      steps {
        container('helm') {
          script {
            def resolveNodeIP = {
              def ip1 = sh(script: 'minikube ip 2>/dev/null || true', returnStdout: true).trim()
              if (ip1) { return ip1 }
              def ip2 = sh(script: "kubectl get node -o jsonpath='{.items[0].status.addresses[?(@.type==\"InternalIP\")].address}' 2>/dev/null || true", returnStdout: true).trim()
              if (ip2) { return ip2 }
              def ip3 = sh(script: '''kubectl get nodes -o wide | awk 'NR==2 {print $6}' 2>/dev/null || true''', returnStdout: true).trim()
              if (ip3) { return ip3 }
              return ''
            }
            if (params.REGISTRY_HOST_OVERRIDE?.trim()) {
              env.REGISTRY_HOST = params.REGISTRY_HOST_OVERRIDE.trim()
              echo "Using REGISTRY_HOST_OVERRIDE: ${env.REGISTRY_HOST}"
            } else {
              def nodeIp = resolveNodeIP()
              if (!nodeIp) { error 'Could not resolve Minikube / node InternalIP (tried minikube ip, jsonpath, wide output).' }
              env.REGISTRY_HOST = nodeIp + ':' + params.REGISTRY_NODEPORT
              echo "Resolved Node IP: ${nodeIp}"
            }
            if (!env.REGISTRY_HOST || env.REGISTRY_HOST == 'unset') {
              error "Registry host not resolved (value='${env.REGISTRY_HOST}')"
            }
            echo "Final REGISTRY_HOST: ${env.REGISTRY_HOST}"
            echo "Reminder: mark ${env.REGISTRY_HOST} insecure inside minikube containerd if pulls fail with HTTPS attempts.";
          }
        }
      }
    }

    stage('Mirror Base Image') {
      steps {
        container('kaniko') {
          sh """
cat > Dockerfile.mirror <<'EOF'
FROM ${params.UPSTREAM_BASE_IMAGE}
LABEL mirror.from=${params.UPSTREAM_BASE_IMAGE}
EOF
/kaniko/executor \
  --context=${WORKSPACE} \
  --dockerfile=${WORKSPACE}/Dockerfile.mirror \
  --destination=${REGISTRY_HOST}/${params.MIRRORED_BASE_IMAGE} \
  --insecure --insecure-pull
"""
        }
      }
    }

    stage('Build & Test') {
      steps { container('maven') { sh 'mvn -B test' } }
    }

    stage('Package & Dockerfile') {
      steps {
        container('maven') {
          sh 'mvn -B -DskipTests package'
          writeFile file: 'Dockerfile', text: """
FROM ${REGISTRY_HOST}/${params.MIRRORED_BASE_IMAGE}
WORKDIR /app
COPY target/*SNAPSHOT.jar app.jar
EXPOSE 8080 8081
ENTRYPOINT [\"java\",\"-jar\",\"/app/app.jar\"]
"""
        }
      }
    }

    stage('Build Image (Kaniko)') {
      steps { container('kaniko') { sh '/kaniko/executor --context=${WORKSPACE} --dockerfile=${WORKSPACE}/Dockerfile --destination=${REGISTRY_HOST}/${APP_NAME}:latest --insecure --insecure-pull' } }
    }

    stage('Resolve Chart') {
      steps {
        container('helm') {
          script {
            def defaultPath = "${WORKSPACE}/${params.CHART_PATH}"
            if (fileExists(defaultPath)) {
              env.CHART_DIR = defaultPath
            } else {
              def found = sh(script: "find ${WORKSPACE} -maxdepth 5 -type f -name Chart\\.yaml -print | head -1 || true", returnStdout: true).trim()
              if (found) {
                env.CHART_DIR = sh(script: "dirname ${found}", returnStdout: true).trim()
              } else {
                error "Helm chart not found at ${defaultPath}; provide CHART_PATH param or add chart to repo"
              }
            }
            echo "Using Helm chart directory: ${env.CHART_DIR}"
          }
        }
      }
    }

    stage('Verify Namespace') {
      steps {
        container('helm') {
          sh 'echo "(Non-blocking) Checking namespace access..."'
          sh 'kubectl get ns ${KUBE_NAMESPACE} 2>&1 || echo "Namespace check skipped / insufficient RBAC, continuing."'
        }
      }
    }

    stage('Deploy (Helm)') {
      steps { container('helm') { sh 'helm upgrade --install ' + params.APP_NAME + ' ${CHART_DIR} -n ' + KUBE_NAMESPACE + ' --set app.name=' + params.APP_NAME + ' --set image.repository=' + '${REGISTRY_HOST}/' + params.APP_NAME + ' --set image.tag=latest --set image.pullPolicy=IfNotPresent' } }
    }
  }

  post {
    success { echo 'Deployment successful.' }
    failure { echo 'Pipeline failed.' }
    always  { echo "Result: ${currentBuild.currentResult}" }
  }
}
