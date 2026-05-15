pipeline {
    agent any
    
    environment {
        // Docker Hub ID와 리포지토리 이름을 설정하세요
        DOCKERHUB_REPO = "aransword/test"
        DOCKERHUB_CREDENTIALS_ID = "DOCKERHUB_CREDENTIALS" // Jenkins에 등록한 ID
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

        // --- 추가된 부분 시작 ---
        stage('Docker Build & Push') {
            steps {
                script {
                    // Docker Hub 로그인 및 Push를 안전하게 처리하기 위해 withDockerRegistry 사용
                    docker.withRegistry('', DOCKERHUB_CREDENTIALS_ID) {
                        // 1. 이미지 빌드
                        def customImage = docker.build("${DOCKERHUB_REPO}:${IMAGE_TAG}")
                        
                        // 2. 이미지 Push
                        customImage.push()
                        
                        // (선택사항) latest 태그로도 push하고 싶다면
                        customImage.push("latest")
                    }
                }
            }
        }
        // --- 추가된 부분 끝 ---
    }
    
    post {
        always {
            // 빌드 완료 후 로컬에 남은 이미지 삭제 (디스크 용량 관리)
            sh "docker rmi ${DOCKERHUB_REPO}:${IMAGE_TAG} || true"
        }
    }
}
