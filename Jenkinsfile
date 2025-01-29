pipeline {
    agent { label 'prod' }

    environment {
        PORTAINER_SERVER_URL = "${env.PORTAINER_SERVER_URL}"
        PORTAINER_TOKEN = credentials('PORTAINER_TOKEN') // Aquí no se hace interpolación
        DOCKERHUB_CREDENTIALS = credentials('DOCKERHUB_CREDENTIALS')
        CONTAINER_NAME = 'webpage'
        IMAGE_NAME = 'albertomoran/webpage:latest'
        ENVIRONMENT_ID = '6'
    }

    stages {
        stage('Build docker image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}")
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_CREDENTIALS', usernameVariable: 'DOCKERHUB_USR', passwordVariable: 'DOCKERHUB_PSW')]) {
                        sh """
                        echo "${DOCKERHUB_PSW}" | docker login -u "${DOCKERHUB_USR}" --password-stdin
                        docker push ${IMAGE_NAME}
                        docker logout
                        """
                    }
                }
            }
        }

        stage('Deploy on Portainer') {
            steps {
                script {
                    // Enviar la solicitud sin interpolación de la variable PORTAINER_TOKEN
                    def checkResponse = httpRequest(
                        url: "${PORTAINER_SERVER_URL}/endpoints/${ENVIRONMENT_ID}/docker/containers/${CONTAINER_NAME}/json",
                        httpMode: 'GET',
                        customHeaders: [[name: 'X-API-Key', value: PORTAINER_TOKEN]],
                        validResponseCodes: '200:404'
                    )
                    
                    if (checkResponse.status == 200) {
                        def containerInfo = new groovy.json.JsonSlurper().parseText(checkResponse.content)
                    
                        echo "Removing container ${CONTAINER_NAME}..."
                        httpRequest(
                            url: "${PORTAINER_SERVER_URL}/endpoints/${ENVIRONMENT_ID}/docker/containers/${containerInfo.Id}?force=true",
                            httpMode: 'DELETE',
                            customHeaders: [[name: 'X-API-Key', value: PORTAINER_TOKEN]],
                            validResponseCodes: '200:204'
                        )
                    }

                    // Deploy the new container
                    def deployConfig = """
                    {
                        "Name": "${CONTAINER_NAME}",
                        "Image": "${IMAGE_NAME}",
                        "ExposedPorts": {
                            "80/tcp": {}
                        },
                        "HostConfig": {
                            "PortBindings": {
                                "80/tcp": [
                                    { "HostPort": "8080", "HostIp": "0.0.0.0" }
                                ]
                            },
                            "RestartPolicy": {
                                "Name": "always"
                            }
                        }
                    }
                    """

                    def createResponse = httpRequest(
                        url: "${PORTAINER_SERVER_URL}/endpoints/${ENVIRONMENT_ID}/docker/containers/create?name=${CONTAINER_NAME}",
                        httpMode: 'POST',
                        contentType: 'APPLICATION_JSON',
                        customHeaders: [[name: 'X-API-Key', value: PORTAINER_TOKEN]],
                        requestBody: deployConfig,
                        validResponseCodes: '200:201'
                    )

                    if (createResponse.status == 201) {
    echo "Container ${CONTAINER_NAME} created successfully."
} else {
    echo "Failed to create container ${CONTAINER_NAME}. Response: ${createResponse.status}"
    echo createResponse.content
}

                    def containerId = new groovy.json.JsonSlurper().parseText(createResponse.content).Id
                    
                    echo "Container ID: ${containerId}"

                    // Start the new container
                    httpRequest(
                        url: "${PORTAINER_SERVER_URL}/endpoints/${ENVIRONMENT_ID}/docker/containers/${containerId}/start",
                        httpMode: 'POST',
                        customHeaders: [[name: 'X-API-Key', value: PORTAINER_TOKEN]],
                        validResponseCodes: '200:204'
                    )
                }
            }
        }
    }
}
