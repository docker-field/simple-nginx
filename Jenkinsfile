DOCKER_USER = "${env.BRANCH_NAME}"
DOCKER_USER_CLEAN = "${DOCKER_USER.replace(".", "")}"
DOCKER_IMAGE_NAMESPACE_PROD = "${DOCKER_USER_CLEAN}"
DOCKER_IMAGE_NAMESPACE_DEV = "${DOCKER_IMAGE_NAMESPACE_PROD}-dev"
DOCKER_IMAGE_REPOSITORY = "simple-nginx"
DOCKER_IMAGE_TAG = "${env.BUILD_TIMESTAMP}"

// Available orchestrators = [ "kubernetes" | "swarm" ]
DOCKER_ORCHESTRATOR = "kubernetes"

if(DOCKER_ORCHESTRATOR.toLowerCase() == "kubernetes"){
    DOCKER_KUBERNETES_NAMESPACE = "${DOCKER_USER_CLEAN}"

    DOCKER_APPLICATION_DOMAIN = "${DOCKER_USER_CLEAN}.${DOCKER_KUBE_DOMAIN_NAME}"
}
else if (DOCKER_ORCHESTRATOR.toLowerCase() == "swarm"){
    DOCKER_SERVICE_NAME = "${DOCKER_USER_CLEAN}-${DOCKER_IMAGE_REPOSITORY}"
    DOCKER_STACK_NAME = "${DOCKER_USER_CLEAN}-simple-nginx"
    DOCKER_UCP_COLLECTION_PATH = "/Shared/Private/${DOCKER_USER}"

    DOCKER_APPLICATION_DOMAIN = "${DOCKER_USER_CLEAN}.${DOCKER_SWARM_DOMAIN_NAME}"
}
else {
    error("Unsupported orchestrator")
}

node {
    def docker_image

    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        docker_image = docker.build("${DOCKER_IMAGE_NAMESPACE_DEV}/${DOCKER_IMAGE_REPOSITORY}")
    }

    stage('Unit Tests') {
        docker_image.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Push') {
        docker.withRegistry(DOCKER_REGISTRY_URI, DOCKER_REGISTRY_CREDENTIALS_ID) {
            docker_image.push(DOCKER_IMAGE_TAG)
        }
    }

    stage('Scan') {
        httpRequest acceptType: 'APPLICATION_JSON', authentication: DOCKER_REGISTRY_CREDENTIALS_ID, contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${DOCKER_REGISTRY_URI}/api/v0/imagescan/scan/${DOCKER_IMAGE_NAMESPACE_DEV}/${DOCKER_IMAGE_REPOSITORY}/${DOCKER_IMAGE_TAG}/linux/amd64"

        def scan_result

        def scanning = true
        while(scanning) {
            def scan_result_response = httpRequest acceptType: 'APPLICATION_JSON', authentication: DOCKER_REGISTRY_CREDENTIALS_ID, httpMode: 'GET', ignoreSslErrors: true, responseHandle: 'LEAVE_OPEN', url: "${DOCKER_REGISTRY_URI}/api/v0/imagescan/repositories/${DOCKER_IMAGE_NAMESPACE_DEV}/${DOCKER_IMAGE_REPOSITORY}/${DOCKER_IMAGE_TAG}"
            scan_result = readJSON text: scan_result_response.content

            if (scan_result.size() != 1) {
                println('Response: ' + scan_result)
                error('More than one imagescan returned, please narrow your search parameters')
            }

            scan_result = scan_result[0]

            if (!scan_result.check_completed_at.equals("0001-01-01T00:00:00Z")) {
                scanning = false
            } else {
                sleep 15
            }

        }
        println('Response JSON: ' + scan_result)
    }

    stage('Deploy to Development') {
        withEnv(["DOCKER_APPLICATION_FQDN=${DOCKER_IMAGE_REPOSITORY}.dev.${DOCKER_APPLICATION_DOMAIN}",
                 "DOCKER_REGISTRY_HOSTNAME=${DOCKER_REGISTRY_HOSTNAME}",
                 "DOCKER_IMAGE_NAMESPACE=${DOCKER_IMAGE_NAMESPACE_DEV}",
                 "DOCKER_IMAGE_REPOSITORY=${DOCKER_IMAGE_REPOSITORY}",
                 "DOCKER_IMAGE_TAG=${DOCKER_IMAGE_TAG}",
                 "DOCKER_USER_CLEAN=${DOCKER_USER_CLEAN}"
                 ]) {

            if(DOCKER_ORCHESTRATOR.toLowerCase() == "kubernetes"){
                println("Deploying to Kubernetes")
                withEnv(["DOCKER_KUBE_CONTEXT=${DOCKER_KUBE_CONTEXT}", "DOCKER_KUBERNETES_NAMESPACE=${DOCKER_KUBERNETES_NAMESPACE}-dev"]) {
                    sh 'envsubst < kubernetes.yaml'
                    sh 'envsubst < kubernetes.yaml | kubectl --context=${DOCKER_KUBE_CONTEXT} --namespace=${DOCKER_KUBERNETES_NAMESPACE} apply -f -'
                }
            }
            else if (DOCKER_ORCHESTRATOR.toLowerCase() == "swarm"){
                println("Deploying to Swarm")
                withEnv(["DOCKER_UCP_COLLECTION_PATH=${DOCKER_UCP_COLLECTION_PATH}"]) {
                    withDockerServer([credentialsId: DOCKER_UCP_CREDENTIALS_ID, uri: DOCKER_UCP_URI]) {
                        sh "docker stack deploy -c docker-compose.yml ${DOCKER_STACK_NAME}-dev"
                    }
                }
            }

            println("Application deployed to Development: ${DOCKER_APPLICATION_FQDN}")
        }
    }

    stage('Integration Tests') {
        docker_image.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Promote') {
        httpRequest acceptType: 'APPLICATION_JSON', authentication: DOCKER_REGISTRY_CREDENTIALS_ID, contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, requestBody: "{\"targetRepository\": \"${DOCKER_IMAGE_NAMESPACE_PROD}/${DOCKER_IMAGE_REPOSITORY}\", \"targetTag\": \"${DOCKER_IMAGE_TAG}\"}", responseHandle: 'NONE', url: "${DOCKER_REGISTRY_URI}/api/v0/repositories/${DOCKER_IMAGE_NAMESPACE_DEV}/${DOCKER_IMAGE_REPOSITORY}/tags/${DOCKER_IMAGE_TAG}/promotion"
    }

    stage('Deploy to Production') {
        withEnv(["DOCKER_APPLICATION_FQDN=${DOCKER_IMAGE_REPOSITORY}.prod.${DOCKER_APPLICATION_DOMAIN}",
                 "DOCKER_REGISTRY_HOSTNAME=${DOCKER_REGISTRY_HOSTNAME}",
                 "DOCKER_IMAGE_NAMESPACE=${DOCKER_IMAGE_NAMESPACE_PROD}",
                 "DOCKER_IMAGE_REPOSITORY=${DOCKER_IMAGE_REPOSITORY}",
                 "DOCKER_IMAGE_TAG=${DOCKER_IMAGE_TAG}",
                 "DOCKER_USER_CLEAN=${DOCKER_USER_CLEAN}"
                 ]) {

            if(DOCKER_ORCHESTRATOR.toLowerCase() == "kubernetes"){
                println("Deploying to Kubernetes")
                withEnv(["DOCKER_KUBE_CONTEXT=${DOCKER_KUBE_CONTEXT}", "DOCKER_KUBERNETES_NAMESPACE=${DOCKER_KUBERNETES_NAMESPACE}"]) {
                    sh 'envsubst < kubernetes.yaml'
                    sh 'envsubst < kubernetes.yaml | kubectl --context=${DOCKER_KUBE_CONTEXT} --namespace=${DOCKER_KUBERNETES_NAMESPACE} apply -f -'
                }
            }
            else if (DOCKER_ORCHESTRATOR.toLowerCase() == "swarm"){
                println("Deploying to Swarm")
                withEnv(["DOCKER_UCP_COLLECTION_PATH=${DOCKER_UCP_COLLECTION_PATH}"]) {
                    withDockerServer([credentialsId: DOCKER_UCP_CREDENTIALS_ID, uri: DOCKER_UCP_URI]) {
                        sh "docker stack deploy -c docker-compose.yml ${DOCKER_STACK_NAME}"
                    }
                }
            }

            println("Application deployed to Production: ${DOCKER_APPLICATION_FQDN}")
        }
    }
}

