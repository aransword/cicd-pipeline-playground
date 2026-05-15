pipeline {
    agent any
    
    environment {
        // Docker Hub ID와 리포지토리 이름
        DOCKERHUB_REPO = "aransword/test"
        DOCKERHUB_CREDENTIALS_ID = "DOCKERHUB_CREDENTIALS" // Jenkins에 등록한 ID (PAT 사용)
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    triggers {
        GenericTrigger(
            genericVariables: [
                [key: 'PR_ACTION', value: '$.action']
            ],
            token: 'my-pr-close-token',
            regexpFilterText: '$PR_ACTION',
            regexpFilterExpression: '^closed$',
            causeString: 'Triggered by GitHub PR Closed Event'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonartest') {
                    sh 'chmod +x gradlew'
                    // 빌드와 분석
                    sh './gradlew clean build sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // --- Jib을 활용한 Build & Push ---
        stage('Jib Build & Push to Docker Hub') {
            steps {
                // Jenkins Credentials에 저장된 ID/PW(또는 PAT)를 환경변수로 꺼내옵니다.
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKERHUB_CREDENTIALS_ID, 
                    usernameVariable: 'DOCKER_USER', 
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    // Gradle Jib task 실행 (파라미터로 계정 정보와 이미지 이름 전달)
                    sh '''
                    ./gradlew jib \
                        -Djib.to.image=docker.io/$DOCKERHUB_REPO:$IMAGE_TAG \
                        -Djib.to.auth.username=$DOCKER_USER \
                        -Djib.to.auth.password=$DOCKER_PASS \
                        -Djib.to.tags=latest
                    '''
                }
            }
        }
    }
}
