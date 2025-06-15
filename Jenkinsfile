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
                    def compareTarget = ""
                    if (env.BRANCH_NAME == 'main') {
                       compareTarget = "HEAD~1"
                    } else {
                        sh "git fetch origin main"
                        compareTarget = "origin/main"
                    }
                    sh "git fetch origin main:refs/remotes/origin/main"
                    def changes = sh(script: "git diff --name-only  ${compareTarget} HEAD", returnStdout: true).trim().split("\n")
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

        stage('Build And Push Docker Images') {
            steps {
                script {
                    def imageTag = ''
                    if (env.TAG_NAME) {
                        imageTag = env.TAG_NAME
                    } else {
                        imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    }
                    def services = []
                    if (env.BRANCH_NAME == 'main' || env.TAG_NAME) {
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
                                --build-arg EXPOSED_PORT=8080 . 
                        """
                        sh "docker push ${env.DOCKER_USER}/${service}:${imageTag}"
                        if (env.BRANCH_NAME == 'main' && !env.TAG_NAME) {
                            sh "docker tag ${env.DOCKER_USER}/${service}:${imageTag} ${env.DOCKER_USER}/${service}:latest"
                            sh "docker push ${env.DOCKER_USER}/${service}:latest"
                        }
                    }
                }
            }
        }
        
         stage('Docker Cleanup And Logout') {
            steps {
                script {
                    echo "Cleaning up Docker and logging out..."
                    sh "docker system prune -af || true"
                    sh "docker logout || true"
                }
            }
        }
        stage ('Push Commit To Helm Repo') {
            when {
                expression { return env.TAG_NAME || env.BRANCH_NAME == 'main' }
            }
                steps {
                    script { 
                        def commit = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()   
                        withCredentials([usernamePassword(credentialsId: 'github-repo-helm', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                            sh '''
                                rm -rf helm
                                git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/minhthien278/project03_k8s_thien helm
                                cd helm
                                git config user.name "jenkins"
                                git config user.email "jenkins@example.com"
                            '''
                        }
                        
                        // Update Chart version
                        sh """
                            cd helm
                            old_version=\$(grep '^version:' Chart.yaml | cut -d' ' -f2)
                            echo "Old version: \$old_version"

                            tmp1=\$(echo "\$old_version" | cut -d. -f1)
                            tmp2=\$(echo "\$old_version" | cut -d. -f2)
                            patch=\$(echo "\$old_version" | cut -d. -f3)
                                
                            new_patch=\$((patch + 1))
                            new_version="\$tmp1.\$tmp2.\$new_patch"
                            echo "New version: \$new_version"

                            # Update version using sed
                            sed -i "s/^version: .*/version: \$new_version/" Chart.yaml
                        """
                        
                        def COMMIT_MESSAGE = ""
                        
                        if (env.TAG_NAME) {
                            echo "Deploying to Kubernetes with tag: ${env.TAG_NAME}"
                            COMMIT_MESSAGE = "Deploy for tag ${env.TAG_NAME}"
                            sh """
                                cd helm
                                sed -i "s/^imageTag: .*/imageTag: \\&tag ${env.TAG_NAME}/" environments/staging/values.yaml
                            """ 
                            echo "Update tag for all services to ${env.TAG_NAME} in environments/staging/values.yaml"
                        } else {
                            echo "Deploying to Kubernetes with branch: main"
                            COMMIT_MESSAGE = "Deploy to helm repo with commit ${commit}"
                            
                            if (env.CHANGED_SERVICES) {
                                env.CHANGED_SERVICES.split(',').each { fullName ->
                                    def shortName = fullName.replaceFirst('spring-petclinic-', '')
                                    sh """
                                        cd helm
                                        sed -i '/${shortName}:/{n;n;s/tag:.*/tag: ${commit}/}' environments/dev/values.yaml
                                    """
                                    echo "Updated tag for ${shortName} to ${commit} in environments/dev/values.yaml"
                                }
                            }
                        }
                        
                        sh """
                            cd helm
                            git add .
                            git commit -m "${COMMIT_MESSAGE}"
                            git push origin main
                        """
                    }
                }
            }
        }
    
    post {
        success {
            echo "CI pipeline finished"
        }
        failure {
            echo "CI pipeline failed!"
        }
    }
}
//Test handel staging