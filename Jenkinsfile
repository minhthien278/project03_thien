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
        ARTIFACT_VERSION = '3.4.1'
    }

    stages {
        stage("Checkout") {
            steps {
                checkout scm
                sh "git fetch --tags"
            }
        }

        stage('Detect Release') {
            steps {
                script {
                    def tagName = sh(script: "git tag --points-at HEAD", returnStdout: true).trim()
                    if (tagName) {
                        echo "Release tag detected: ${tagName}"
                        env.TAG_NAME = tagName
                    }
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

                    def allServices = SERVICES.split("\n").collect { it.trim() }.findAll { it }

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
                withCredentials([usernamePassword(credentialsId: 'docker-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        env.DOCKER_USER = DOCKER_USER
                        sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                    }
                }
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                script {
                    def isMainBranch = (env.BRANCH_NAME == 'main' || env.GIT_BRANCH == 'origin/main')
                    def imageTag = ''
                    if (env.TAG_NAME) {
                        imageTag = env.TAG_NAME
                    } else if (isMainBranch) {
                        imageTag = 'latest'
                    } else {
                        imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    }
                    env.IMAGE_TAG = imageTag
                    def services = []
                    if (isMainBranch || env.TAG_NAME) {
                        services = SERVICES.split("\n").collect { it.trim() }.findAll { it }
                    } else {
                        services = env.CHANGED_SERVICES.split(',').collect { it.trim() }
                    }

                    if (!services || services.isEmpty()) {
                        echo "No services to build. Skipping."
                        return
                    }

                    for (service in services) {
                        echo "Building and pushing image for ${service}:${imageTag}"

                        sh "./mvnw clean install -pl ${service} -DskipTests"

                        sh """
                            cd ${service} && \\
                            docker build \\
                                -t ${env.DOCKER_USER}/${service}:${imageTag} \\
                                -f ../docker/Dockerfile \\
                                --build-arg ARTIFACT_NAME=target/${service}-${ARTIFACT_VERSION} \\
                                --build-arg EXPOSED_PORT=8080 . && \\
                            docker push ${env.DOCKER_USER}/${service}:${imageTag}
                        """
                    }
                }
            }
        }

        stage('Docker Cleanup and Logout') {
            steps {
                script {
                    echo "Cleaning up Docker and logging out..."
                    sh "docker system prune -af || true"
                    sh "docker logout || true"
                }
            }
        }
    }

    post {
        success {
            echo "CI pipeline finished. Built tag: ${env.IMAGE_TAG}"
        }
        failure {
            echo "CI pipeline failed!"
        }
    }
}
