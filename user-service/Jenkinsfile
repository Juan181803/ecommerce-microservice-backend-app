pipeline {
  agent any

  environment {
    K8S_MANIFESTS_DIR = 'k8s'
    NEWMAN_IMAGE_NAME = 'juanito0702'
    NEWMAN_IMAGE_TAG = "latest"
    NEWMAN_REPORTS_DIR = 'newman-reports'
    PROFILE = 'dev'
    DOCKER_IMAGE_NAME = 'juanito0702'
  }

  stages {

    stage('Preparar entorno') {
      steps {
        sh '''
          echo "Instalando dependencias necesarias..."
          apt-get update
          apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
          
          echo "Limpiando configuración anterior de Docker..."
          rm -f /etc/apt/sources.list.d/docker.list
          
          echo "Configurando repositorio de Docker..."
          install -m 0755 -d /etc/apt/keyrings
          curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
          chmod a+r /etc/apt/keyrings/docker.gpg
          
          echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
          
          echo "Actualizando repositorios..."
          apt-get update
          
          echo "Instalando Docker..."
          apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
          
          echo "Verificando instalación de Docker..."
          docker --version
        '''
      }
    }

    stage('User Input') {
      steps {
        script {
          env.PROFILE = input message: "Elige un perfil (prod / dev / stage)",
                              parameters: [choice(name: 'PROFILE', choices: ['prod', 'dev', 'stage'], description: 'Selecciona el perfil de despliegue')]
        }
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    
    stage('Build') {
      steps {
        sh '''
          echo "Building the project..."
          docker run --rm -v "$WORKSPACE":/app -w /app maven:3.9.6-eclipse-temurin-11 mvn clean package -DskipTests
        '''
      }
    }

    stage('Unit and Integration Tests') {
      steps {
        sh '''
          echo "Running unit and integration tests..."
          docker run --rm -v "$WORKSPACE":/app -w /app maven:3.9.6-eclipse-temurin-11 mvn clean verify -DskipTests=false
        '''
      }
    }

    stage('Build and Push Docker Images') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'docker-hub-credentials',
          usernameVariable: 'DOCKER_USERNAME',
          passwordVariable: 'DOCKER_PASSWORD'
        )]) {
          sh '''
            echo "Logging in to Docker Hub..."
            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

            echo "Building and pushing Docker images..."
            docker-compose -f compose.yml build
            docker-compose -f compose.yml push

            echo "Logout from Docker Hub..."
            docker logout
          '''
        }
      }
    }
    
    stage('Desplegar manifiestos') {
      steps {
        sh '''
          echo "Deploying Core Services..."
          echo "Deploying Zipkin..."
          kubectl apply -f ${K8S_MANIFESTS_DIR}/core/zipkin-deployment.yaml
          kubectl wait --for=condition=ready pod -l app=zipkin --timeout=200s
          
          echo "Deploying Service Discovery..."
          kubectl apply -f ${K8S_MANIFESTS_DIR}/$PROFILE/service-discovery-deployment.yaml
          kubectl wait --for=condition=ready pod -l app=service-discovery --timeout=300s

          echo "Deploying Cloud Config..."
          kubectl apply -f ${K8S_MANIFESTS_DIR}/core/cloud-config-deployment.yaml
          kubectl wait --for=condition=ready pod -l app=cloud-config --timeout=300s
        '''
      }
    }

    stage('Desplegar microservicios') {
      steps {
        sh """
          echo "Deploying Microservices..."
          kubectl apply -f ${K8S_MANIFESTS_DIR}/${PROFILE}/api-gateway-deployment.yaml
          kubectl wait --for=condition=ready pod -l app=api-gateway --timeout=300s
          kubectl apply -f ${K8S_MANIFESTS_DIR}/${PROFILE}/favourite-service-deployment.yaml
          kubectl wait --for=condition=ready pod -l app=favourite-service --timeout=300s
          kubectl apply -f ${K8S_MANIFESTS_DIR}/${PROFILE}/order-service-deployment.yaml
          kubectl wait --for=condition=ready pod -l app=order-service --timeout=300s
          kubectl apply -f ${K8S_MANIFESTS_DIR}/${PROFILE}/payment-service-deployment.yaml
          kubectl wait --for=condition=ready pod -l app=payment-service --timeout=300s
          kubectl apply -f ${K8S_MANIFESTS_DIR}/${PROFILE}/product-service-deployment.yaml
          kubectl wait --for=condition=ready pod -l app=product-service --timeout=300s
          kubectl apply -f ${K8S_MANIFESTS_DIR}/${PROFILE}/proxy-client-deployment.yaml
          kubectl wait --for=condition=ready pod -l app=proxy-client --timeout=300s
          kubectl apply -f ${K8S_MANIFESTS_DIR}/${PROFILE}/shipping-service-deployment.yaml
          kubectl wait --for=condition=ready pod -l app=shipping-service --timeout=300s
          kubectl apply -f ${K8S_MANIFESTS_DIR}/${PROFILE}/user-service-deployment.yaml
          kubectl wait --for=condition=ready pod -l app=user-service --timeout=300s
        """
      }
    }
    stage('Correr e2e') {
      when {
        expression { env.PROFILE == 'stage' }
      }
      steps {
        sh '''
          echo "Running E2E tests..."
          kubectl apply -f ${K8S_MANIFESTS_DIR}/core/newman-e2e-job.yaml
          kubectl wait --for=condition=complete job/newman-e2e-job --timeout=600s
          echo "Fetching Newman results..."
          kubectl logs job/newman-e2e-tests
        '''
      }
    }
    
    stage('Desplegar Locust') {
      when {
        expression { env.PROFILE == 'dev' || env.PROFILE == 'stage' }
      }
      steps {
        sh '''
          echo "Deploying Locust..."
          kubectl apply -f ${K8S_MANIFESTS_DIR}/core/locust-deployment.yaml

          echo "Esperando a que el servicio esté disponible..."
          kubectl wait --for=condition=ready pod -l app=locust --timeout=300s
          
          echo "Obteniendo la URL de Locust..."
          LOCUST_URL=$(kubectl get svc locust -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
          if [ -z "$LOCUST_URL" ]; then
            LOCUST_URL="localhost"
          fi
          echo "Locust está disponible en: http://$LOCUST_URL:8089"
        '''
      }
    }
    
  }
}