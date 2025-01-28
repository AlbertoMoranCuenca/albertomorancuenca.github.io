pipeline {
    agent { label 'prod' }  

    environment {
        PORTAINER_SERVER_URL = "${env.PORTAINER_SERVER_URL}"
        PORTAINER_TOKEN = credentials('PORTAINER_TOKEN')
        CONTAINER_NAME = 'albertomoran-webpage'
        IMAGE_NAME = 'albertomoran-webpage:latest'
        ENVIRONMENT_ID = '6'
    }

    stages {
        stage('Build docker image'){
            steps {
                script {
                    sh """
                    docker build -t ${IMAGE_NAME} .
                    """
                }
            }
        }

        stage('Run Container') {
            steps {
                script {
                    def deployConfig = """
                    {
                        "Name": "${CONTAINER_NAME}",
                        "Env": [],
                        "EndpointId": 6, 
                        "Config": {
                            "Image": "${IMAGE_NAME}",
                            "RestartPolicy": {
                                "Name": "always"
                            },
                            "ExposedPorts": {
                                "80/tcp": {}
                            },
                            "HostConfig": {
                                "PortBindings": {
                                    "80/tcp": [
                                        {
                                            "HostPort": "80"
                                        }
                                    ]
                                }
                            }
                        }
                    }
                    """
                    httpRequest(
                        url: "${PORTAINER_SERVER_URL}/endpoints/${ENVIRONMENT_ID}/docker/containers/create",
                        httpMode: 'POST',
                        contentType: 'APPLICATION_JSON',
                        customHeaders: [
                            [name: 'Authorization', value: "Bearer ${PORTAINER_TOKEN}"]
                        ],
                        requestBody: deployConfig,
                        validResponseCodes: '200:201'
                    )

                    httpRequest(
                        url: "${PORTAINER_SERVER_URL}/endpoints/${ENVIRONMENT_ID}/docker/${CONTAINER_NAME}/start",
                        httpMode: 'POST',
                        customHeaders: [
                            [name: 'Authorization', value: "Bearer ${PORTAINER_TOKEN}"]
                        ],
                        validResponseCodes: '200:204'
                    )
                }
            }
        }
    }
}
