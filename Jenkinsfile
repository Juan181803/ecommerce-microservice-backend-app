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
        stage('Preparación') {
            steps {
                sh '''
                    echo "Verificando Java..."
                    java -version
                    
                    echo "Verificando archivos necesarios..."
                    echo "Current directory: $(pwd)"
                    echo "Workspace directory: $WORKSPACE"
                    echo "Listing all files in current directory:"
                    ls -la
                    
                    if [ ! -f "pom.xml" ]; then
                        echo "ERROR: pom.xml no encontrado en $(pwd)"
                        echo "Buscando pom.xml en el workspace..."
                        find $WORKSPACE -name "pom.xml"
                        exit 1
                    fi

                    echo "Verificando Maven wrapper..."
                    if [ ! -f "mvnw" ]; then
                        echo "ERROR: mvnw no encontrado"
                        exit 1
                    fi
                    chmod +x mvnw
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
        
        stage('Build') {
            steps {
                sh '''
                    echo "Building the project..."
                    echo "Current directory: $(pwd)"
                    echo "Workspace directory: $WORKSPACE"
                    echo "Listing all files in current directory:"
                    ls -la
                    
                    # Compilamos el proyecto
                    export MAVEN_OPTS="-Dmaven.compiler.source=11 -Dmaven.compiler.target=11"
                    ./mvnw clean install -DskipTests -Dlombok.version=1.18.22
                '''
            }
        }
        
        stage('Unit and Integration Tests') {
            steps {
                sh '''
                    echo "Running unit and integration tests for all modules..."
                    ./mvnw verify -Dmaven.compiler.source=11 -Dmaven.compiler.target=11 -Dlombok.addLombokGeneratedAnnotation=true -Dlombok.extern.findbugs.addSuppressFBWarnings=true
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
                    echo "Verificando acceso a Kubernetes..."
                    if ! kubectl cluster-info &>/dev/null; then
                        echo "ERROR: No se puede acceder al cluster de Kubernetes"
                        exit 1
                    fi

                    echo "Deploying Core Services..."
                    echo "Deploying Zipkin..."
                    kubectl apply -f ${K8S_MANIFESTS_DIR}/core/zipkin-deployment.yaml || exit 1
                    kubectl wait --for=condition=ready pod -l app=zipkin --timeout=200s || exit 1
                    
                    echo "Deploying Service Discovery..."
                    kubectl apply -f ${K8S_MANIFESTS_DIR}/$PROFILE/service-discovery-deployment.yaml || exit 1
                    kubectl wait --for=condition=ready pod -l app=service-discovery --timeout=300s || exit 1

                    echo "Deploying Cloud Config..."
                    kubectl apply -f ${K8S_MANIFESTS_DIR}/core/cloud-config-deployment.yaml || exit 1
                    kubectl wait --for=condition=ready pod -l app=cloud-config --timeout=300s || exit 1
                '''
            }
        }

        stage('Desplegar microservicios') {
            steps {
                sh """
                    echo "Deploying Microservices..."
                    for service in api-gateway favourite-service order-service payment-service product-service proxy-client shipping-service user-service; do
                        echo "Deploying $service..."
                        kubectl apply -f ${K8S_MANIFESTS_DIR}/${PROFILE}/${service}-deployment.yaml || exit 1
                        kubectl wait --for=condition=ready pod -l app=${service} --timeout=300s || exit 1
                    done
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
                    kubectl apply -f ${K8S_MANIFESTS_DIR}/core/newman-e2e-job.yaml || exit 1
                    kubectl wait --for=condition=complete job/newman-e2e-job --timeout=600s || exit 1
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
                    kubectl apply -f ${K8S_MANIFESTS_DIR}/core/locust-deployment.yaml || exit 1

                    echo "Esperando a que el servicio esté disponible..."
                    kubectl wait --for=condition=ready pod -l app=locust --timeout=300s || exit 1
                    
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
    
    post {
        always {
            echo "Pipeline finalizado"
        }
        success {
            echo "Pipeline completado exitosamente"
        }
        failure {
            echo "Pipeline falló"
        }
    }
}
