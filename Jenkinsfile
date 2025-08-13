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
    string(name: 'REGISTRY_HOST_OVERRIDE', defaultValue: '', description: 'Optional full registry host:port (defaults to Minikube addon registry)')
    booleanParam(name: 'USE_MINIKUBE_REGISTRY', defaultValue: true, description: 'Auto-detect and use Minikube registry addon service')
    string(name: 'MINIKUBE_REGISTRY_NAMESPACE', defaultValue: 'kube-system', description: 'Namespace of Minikube registry addon')
    string(name: 'MINIKUBE_REGISTRY_SERVICE', defaultValue: 'registry', description: 'Service name of Minikube registry addon')
    string(name: 'REGISTRY_PORT_OVERRIDE', defaultValue: '', description: 'Optional service port to use (detect if empty)')
  }

  environment {
    KUBE_NAMESPACE = 'apps'
    // Default to Minikube registry addon service
    REGISTRY_HOST  = 'registry.kube-system.svc.cluster.local:5000'
    REGISTRY_PORT  = ''
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Resolve Registry Host') {
      steps {
        container('helm') {
          script {
            if (params.REGISTRY_HOST_OVERRIDE?.trim()) {
              env.REGISTRY_HOST = params.REGISTRY_HOST_OVERRIDE.trim()
              echo "Using REGISTRY_HOST_OVERRIDE: ${env.REGISTRY_HOST}"
            } else if (params.USE_MINIKUBE_REGISTRY) {
              def ns = params.MINIKUBE_REGISTRY_NAMESPACE
              def svc = params.MINIKUBE_REGISTRY_SERVICE
              // Probe service JSON only once
              def svcJson = sh(script: "kubectl get svc ${svc} -n ${ns} -o json 2>/dev/null || true", returnStdout: true).trim()
              if (!svcJson) {
                error "Minikube registry service ${svc}.${ns} not found. Enable with: minikube addons enable registry"
              }
              if (params.REGISTRY_PORT_OVERRIDE?.trim()) {
                env.REGISTRY_PORT = params.REGISTRY_PORT_OVERRIDE.trim()
              } else {
                // Try to pick a port mapping to container 5000; fallback first port
                def port = sh(script: "kubectl get svc ${svc} -n ${ns} -o jsonpath='{.spec.ports[?(@.targetPort==5000)].port}' 2>/dev/null || true", returnStdout: true).trim()
                if (!port) {
                  port = sh(script: "kubectl get svc ${svc} -n ${ns} -o jsonpath='{.spec.ports[0].port}' 2>/dev/null || true", returnStdout: true).trim()
                }
                env.REGISTRY_PORT = port ?: '5000'
              }
              def host = "${svc}.${ns}.svc.cluster.local:${env.REGISTRY_PORT}"
              env.REGISTRY_HOST = host
              echo "Resolved Minikube addon registry host: ${env.REGISTRY_HOST}"
              echo "(If pulls fail due to HTTPS attempt, mark it insecure in containerd or use REGISTRY_HOST_OVERRIDE with NodeIP:NodePort + hosts.toml)"
            } else {
              echo "Using default static REGISTRY_HOST=${env.REGISTRY_HOST} (no auto-detect)."
            }
          }
        }
      }
    }

    stage('Mirror Base Image') {
      steps {
        container('kaniko') {
          script {
            if (!params.MIRRORED_BASE_IMAGE?.trim()) {
              error 'MIRRORED_BASE_IMAGE parameter is empty'
            }
            echo "Mirroring ${params.UPSTREAM_BASE_IMAGE} -> ${env.REGISTRY_HOST}/${params.MIRRORED_BASE_IMAGE} (insecure)"
          }
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
          // Fail-fast if registry host still default placeholder but auto-detect requested
          script {
            if (params.USE_MINIKUBE_REGISTRY && !params.REGISTRY_HOST_OVERRIDE?.trim() && !env.REGISTRY_HOST.contains(params.MINIKUBE_REGISTRY_SERVICE)) {
              error "Registry host did not resolve to expected Minikube service (${params.MINIKUBE_REGISTRY_SERVICE}); current=${env.REGISTRY_HOST}"
            }
          }
          writeFile file: 'Dockerfile', text: """
FROM ${REGISTRY_HOST}/${params.MIRRORED_BASE_IMAGE}
WORKDIR /app
COPY target/*SNAPSHOT.jar app.jar
EXPOSE 8080 8081
ENTRYPOINT [\"java\",\"-jar\",\"/app/app.jar\"]
"""
          sh 'echo DOCKERFILE BASE IMAGE: && grep "^FROM" Dockerfile'
          echo "Generated Dockerfile using base image ${REGISTRY_HOST}/${params.MIRRORED_BASE_IMAGE}";
        }
      }
    }

    stage('Build Image (Kaniko)') {
      steps {
        container('kaniko') {
          sh '''
/kaniko/executor \
  --context=${WORKSPACE} \
  --dockerfile=${WORKSPACE}/Dockerfile \
  --destination=${REGISTRY_HOST}/${APP_NAME}:latest \
  --insecure --insecure-pull
'''
        }
      }
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
      steps {
        container('helm') {
          sh 'echo PWD=$(pwd) WORKSPACE=${WORKSPACE} && ls -1 ${WORKSPACE} || true'
          sh 'helm upgrade --install ' + params.APP_NAME + ' ' + '${CHART_DIR}' + ' -n ' + KUBE_NAMESPACE + ' ' + \
             '--set app.name=' + params.APP_NAME + ' --set image.repository=' + '${REGISTRY_HOST}/' + params.APP_NAME + ' --set image.tag=latest --set image.pullPolicy=IfNotPresent'
          echo "Deployed image ${REGISTRY_HOST}/${params.APP_NAME}:latest"
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
