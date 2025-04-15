pipeline {
    agent any
    tools {
        maven "Maven_3.9.9"
    }

    environment {
        DOCKER_HUL_ID = "hulbo"
        IMAGE_NAME = "sc_apigatewayservice"
        PORT = "8000:8000"
    }

    stages {
        stage('Clean dangling Docker images') {
            steps {
                sh 'docker image prune -f'
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Set Spring Profile') {
            steps {
                script {
                    def branch = sh(script: "git branch -r --contains HEAD | grep origin/ | head -n 1 | sed 's@origin/@@'", returnStdout: true).trim()
                    def profile = 'local'
                    if (branch == 'main') {
                        profile = 'prod'
                    } else if (branch == 'develop') {
                        profile = 'dev'
                    } else if (branch == 'test') {
                        profile = 'test'
                    }

                    def tag = 'latest'

                    env.BRANCH = branch
                    env.ACTIVE_PROFILE = profile
                    env.IMAGE_TAG = tag
                    env.FULL_IMAGE_NAME = "${env.DOCKER_HUL_ID}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                    echo "▶ 선택된 브랜치: ${env.BRANCH}"
                    echo "▶ 적용된 Spring Profile: ${env.ACTIVE_PROFILE}"
                    echo "▶ Docker 이미지 태그: ${env.IMAGE_TAG}"
                    echo "▶ Docker 전체 이미지 명: ${env.FULL_IMAGE_NAME}"
                    echo "▶ 환경변수: OCI-SERVICE-PRIVATE: $OCI_SERVICE_PRIVATE"
                }
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def result = sh(script: "docker build -t ${env.FULL_IMAGE_NAME} .", returnStatus: true)
                    if (result != 0) {
                        error "▶ Docker 이미지 빌드 실패!"
                    }
                }
            }
        }

        stage('Deploy to Remote Server') {
            steps {
                script {
                    sh """
                        if docker ps -a --format "{{.Names}}" | grep -q "^${env.IMAGE_NAME}\$"; then
                            docker stop ${env.IMAGE_NAME}
                            docker rm ${env.IMAGE_NAME}
                        fi

                        docker run -d --name ${env.IMAGE_NAME} -p ${env.PORT} -e OCI_SERVICE_PRIVATE=$OCI_SERVICE_PRIVATE -e SPRING_PROFILES_ACTIVE=${env.ACTIVE_PROFILE} ${env.FULL_IMAGE_NAME}

                        docker image prune -f
                    """
                }
            }
        }
    }
}
