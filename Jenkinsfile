pipeline {
    agent any

    environment {
        SERVICES = """
            spring-petclinic-api-gateway
            spring-petclinic-config-server
            spring-petclinic-customers-service
            spring-petclinic-discovery-server
            spring-petclinic-vets-service
            spring-petclinic-visits-service
        """
        DOCKER_USER = 'tlavu2004'
    }

    stages {
        stage("Checkout") {
            steps {
                checkout scm
            }
        }
            stage('Detect Release') {
                steps {
                    script {
                        def tagName = sh(script: "git tag --points-at HEAD", returnStdout: true).trim()
                        echo "Tag name: ${tagName}"
                        env.CHANGED_SERVICES = env.SERVICES
                        env.TAG_NAME = tagName
                        echo "A new release found with tag ${env.TAG_NAME}"
                    }
                }
            }
        stage('Detect Changes') {
            when { expression { return !env.TAG_NAME } }
            steps {
                script {
                    sh "git fetch origin main:refs/remotes/origin/main"
                    def changes = sh(script: "git diff --name-only origin/main HEAD", returnStdout: true).trim().split("\n")
                    echo "Changed files: ${changes}"

                    def allServices = SERVICES.split().collect { it.trim() }

                    def changedServices = allServices.findAll { service ->
                        changes.any { it.contains(service) }
                    }

                    if (changedServices.isEmpty()) {
                        echo "No service changes detected. Skipping build."
                        currentBuild.result = 'SUCCESS'
                        return
                    }

                    echo "Detected changed services: ${changedServices}"
                    env.CHANGED_SERVICES = changedServices.join(',')
                }
            }
        }

        stage('Docker Login') {
            steps {
                sh 'whoami'
                withCredentials([string(credentialsId: 'docker-credentials', variable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }
        stage('Build and push docker image') {
            steps {
                script {
                    def imageTag
                    def services = []
                    if (env.TAG_NAME) {
                        imageTag = env.TAG_NAME
                    } else {
                        imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    }
                    if (env.GIT_BRANCH == 'origin/main' || env.BRANCH_NAME == 'main') { 
                        services = env.SERVICES.split()
                    }
                    else services = env.CHANGED_SERVICES.split(',')

                    for (service in services) {
                        echo "🚀 Building and pushing image for ${service} with tag ${imageTag}"
                        sh "./mvnw clean install -pl ${service} -DskipTests"   
                        sh """
                            cd ${service} && \\
                            docker build \\
                            -t ${DOCKER_USER}/${service}:${imageTag} \\
                            -f ../docker/Dockerfile \\
                            --build-arg ARTIFACT_NAME=target/${service}-3.4.1 \\
                            --build-arg EXPOSED_PORT=8080 .
                            docker push ${DOCKER_USER}/${service}:${imageTag}
                        """
                    }
                }
            }
        }
    }
}
