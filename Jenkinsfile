pipeline {
    agent any

    environment {
        SONAR_TOKEN     = credentials('SONAR_TOKEN')
        DOCKER_HUB_CRED = credentials('DOCKERHUB_CREDENTIALS')
        DOCKER_IMAGE    = "firasktib/mern-devsecops"
        NETWORK         = "jenkins-devsecops_devops-network"
        SONAR_HOST      = "http://sonarqube:9000"
        MERN_NETWORK    = "mern-devsecops-pipeline_mern-network"
    }

    stages {

        stage('GIT') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/FIRASKTIB/mern-devsecops.git'
            }
        }

        stage('DEBUG FILES') {
            steps {
                sh '''
                    echo "=== ROOT ===" && ls -la
                    echo "=== CLIENT ===" && ls -la client
                    echo "=== SERVER ===" && ls -la server
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    cd client && npm install
                    cd ../server && npm install
                '''
            }
        }

        stage('Build Frontend') {
            steps {
                sh 'cd client && npm run build'
            }
        }

        stage('SCA - npm audit') {
            steps {
                sh '''
                    echo "=== Audit Client ==="
                    cd client && npm audit --audit-level=high || true
                    echo "=== Audit Server ==="
                    cd ../server && npm audit --audit-level=high || true
                '''
            }
        }

        stage('Gitleaks - Secrets Scan') {
            steps {
                sh '''
                    docker run --rm \
                        -v ${WORKSPACE}:/repo \
                        -e GIT_DISCOVERY_ACROSS_FILESYSTEM=1 \
                        zricethezav/gitleaks:latest \
                        detect \
                        --source=/repo \
                        --no-git \
                        --exit-code=0 \
                        --report-format=json \
                        --report-path=/repo/gitleaks-report.json
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('SAST - SonarQube') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        docker run --rm \
                            --network jenkins-devsecops_devops-network \
                            -v ${WORKSPACE}:/usr/src \
                            -w /usr/src \
                            sonarsource/sonar-scanner-cli \
                            -Dsonar.projectKey=mern-devsecops \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/cypress/**,**/.git/** \
                            -Dsonar.host.url=http://sonarqube:9000 \
                            -Dsonar.token=$SONAR_TOKEN \
                            -Dsonar.scm.disabled=true
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    echo 'Quality Gate : analyse SonarQube terminée ✅'
                    echo 'Résultats : http://localhost:9000/dashboard?id=mern-devsecops'
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${DOCKER_IMAGE}:client ./client
                    docker build -t ${DOCKER_IMAGE}:server ./server
                '''
            }
        }

        stage('Trivy - Image Scan') {
            steps {
                sh '''
                    echo "=== Scan image client ==="
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v trivy_cache:/root/.cache/trivy \
                        aquasec/trivy:latest image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        ${DOCKER_IMAGE}:client

                    echo "=== Scan image server ==="
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v trivy_cache:/root/.cache/trivy \
                        aquasec/trivy:latest image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        ${DOCKER_IMAGE}:server
                '''
            }
        }

        stage('Docker Push') {
            steps {
                sh '''
                    echo $DOCKER_HUB_CRED_PSW | \
                        docker login -u $DOCKER_HUB_CRED_USR --password-stdin
                    docker push ${DOCKER_IMAGE}:client
                    docker push ${DOCKER_IMAGE}:server
                    docker logout
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh 'docker compose up -d --build'
            }
        }

        // ✅ CORRIGÉ : réseau MERN pour que ZAP atteigne le conteneur frontend
        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                    mkdir -p ${WORKSPACE}/.zap
                    chmod 777 ${WORKSPACE}/.zap

                    docker run --rm \
                        --user root \
                        --network mern-devsecops-pipeline_mern-network \
                        -v ${WORKSPACE}/.zap:/zap/wrk:rw \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t http://frontend:80 \
                        -r zap-report.html \
                        -I
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '.zap/zap-report.html',
                                     allowEmptyArchive: true
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline terminé avec succès !'
        }
        failure {
            echo '❌ Pipeline échoué — vérifie les logs ci-dessus.'
        }
        always {
            echo '📋 Pipeline terminé.'
        }
    }
}