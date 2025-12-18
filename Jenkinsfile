pipeline {
    agent any

 triggers {
        githubPush()
    }

    tools {
        maven 'M2_HOME'
    }

    environment {
        SONAR_HOST_URL = 'http://localhost:9000'
        DOCKER_IMAGE = 'marwenazouzi12/studentmanager:latest'
         KUBECONFIG = '/var/lib/jenkins/.kube/config'
    }

    stages {

        /* ===================== 1️⃣ GIT ===================== */
        stage('Checkout') {
            steps {

 		script {
            // Get branch from webhook or use default
            def branch = env.GIT_BRANCH ?: 'marwen'
            
            // Clean the branch name
            branch = branch.replace('origin/', '')
            branch = branch.replace('refs/heads/', '')
            
            echo "📌 Building branch: ${branch}"
            
            git branch: branch,
                url: 'https://github.com/brouri12/devopswithwebhook.git'
        }            }
        }

        /* ===================== 2️⃣ BUILD ===================== */
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar'
            }
        }

        /* ===================== 3️⃣ SONARQUBE ===================== */
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                        sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=jenkins_sonar \
                        -Dsonar.projectName=student-management \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.tests=src/test/java \
                        -Dsonar.login=${SONAR_TOKEN} \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -DskipTests
                        """
                    }
                }
            }
        }

        /* ===================== 4️⃣ QUALITY GATE ===================== */
        stage('Quality Gate Check') {
    steps {
        script {
            withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                // Wait a bit for SonarQube to process the analysis
                echo "Waiting for SonarQube to process analysis..."
                sleep 30
                
                // Increase timeout to 5 minutes
                timeout(time: 5, unit: 'MINUTES') {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Quality gate failed: ${qg.status}"
                    }
                    echo "✅ Quality gate passed: ${qg.status}"
                }
            }
        }
    }
}

        /* ===================== 5️⃣ DOCKER ===================== */
        stage('Docker Build & Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-registry-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    docker build -t ${DOCKER_IMAGE} .
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push ${DOCKER_IMAGE}
                    '''
                }
            }
        }

        /* ===================== 6️⃣ KUBERNETES ===================== */
       stage('Deploy to Kubernetes') {
    steps {
        sh '''
        kubectl create namespace devops || true

        # Apply MySQL first
        kubectl apply -f k8s/mysql-deployment.yaml -n devops
        

        # Wait for MySQL to be ready
        kubectl wait --for=condition=Ready pod -l app=mysql -n devops --timeout=120s

        # Deploy Spring Boot app
        kubectl apply -f k8s/deployment.yaml -n devops
        kubectl apply -f k8s/service.yaml -n devops
        kubectl rollout status deployment/springboot-app -n devops
        '''
    }
}

    }
}
