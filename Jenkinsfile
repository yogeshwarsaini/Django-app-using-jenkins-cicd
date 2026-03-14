pipeline {
    agent any

    environment {
        // Apna Docker Hub username daalo
        DOCKER_IMAGE = "yogismash/django-app"
        DOCKER_TAG = "${latest}"
        SONAR_PROJECT_KEY = "sonar-key"
    }

    stages {

        // ─────────────────────────────
        // Stage 1: Code Checkout ///
        // ─────────────────────────────
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/yogeshwarsaini/Django-app-using-jenkins-cicd.git'
                echo "✅ Code checkout complete!"
            }
        }

        // ─────────────────────────────
        // Stage 2: SAST - SonarQube //
        // ─────────────────────────────
        // stage('SAST - SonarQube Scan') {
        //     steps {
        //         withSonarQubeEnv('SonarQube') {
        //             sh """
        //                 sonar-scanner \
        //                   -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
        //                   -Dsonar.projectName=${SONAR_PROJECT_KEY} \
        //                   -Dsonar.sources=. \
        //                   -Dsonar.host.url=http://localhost:9000
        //             """
        //         }
        //     }
        // }

        stage('SAST - SonarQube Scan') {
    steps {
        withSonarQubeEnv('SonarQube') {
            sh """
                /opt/sonar-scanner/bin/sonar-scanner \
                  -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                  -Dsonar.projectName=${SONAR_PROJECT_KEY} \
                  -Dsonar.sources=. \
                  -Dsonar.host.url=http://localhost:9000 \
                  -Dsonar.login=sqp_129dc4011621fe129ce5f055537473478f05d15f
            """
        }
    }
}

        // Quality Gate check
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ─────────────────────────────
        // Stage 3: SCA - OWASP DC
        // ─────────────────────────────
        stage('SCA - OWASP Dependency Check') {
            steps {
                sh """
                    /opt/dependency-check/bin/dependency-check.sh \
                      --project "MyApp" \
                      --scan . \
                      --format HTML \
                      --format XML \
                      --out ./reports/dependency-check \
                      --failOnCVSS 7
                """
            }
            post {
                always {
                    // Report publish karo
                    dependencyCheckPublisher \
                        pattern: 'reports/dependency-check/dependency-check-report.xml'
                }
            }
        }

        // ─────────────────────────────
        // Stage 4: Docker Build
        // ─────────────────────────────
        stage('Docker Build') {
            steps {
                sh """
                    docker build \
                      -t ${DOCKER_IMAGE}:${DOCKER_TAG} \
                      -t ${DOCKER_IMAGE}:latest \
                      .
                """
                echo "✅ Docker image built!"
            }
        }

        // ─────────────────────────────
        // Stage 5: Trivy Image Scan
        // ─────────────────────────────
        stage('Trivy Image Scan') {
            steps {
                sh """
                    # Reports folder banao
                    mkdir -p reports/trivy

                    # Image scan karo
                    trivy image \
                      --format table \
                      --output reports/trivy/trivy-report.txt \
                      --severity HIGH,CRITICAL \
                      ${DOCKER_IMAGE}:${DOCKER_TAG}

                    # Report print karo
                    cat reports/trivy/trivy-report.txt
                """
            }
        }

        // ─────────────────────────────
        // Stage 6: Docker Push
        // ─────────────────────────────
        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-token',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh """
                        echo $DOCKER_PASS | \
                        docker login -u $DOCKER_USER --password-stdin

                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:latest

                        echo "✅ Image pushed to Docker Hub!"
                    """
                }
            }
        }

        // ─────────────────────────────
        // Stage 7: Deploy
        // ─────────────────────────────
        stage('Deploy') {
            steps {
                sh """
                    # Purana container stop karo
                    docker stop myapp || true
                    docker rm myapp || true

                    # Naya container run karo
                    docker run -d \
                      --name myapp \
                      --restart always \
                      -p 8090:80 \
                      ${DOCKER_IMAGE}:${DOCKER_TAG}

                    echo "✅ App deployed on port 8090!"
                """
            }
        }
    }

    // Pipeline complete hone ke baad
    post {
        success {
            echo '✅ Pipeline Successful!'
        }
        failure {
            echo '❌ Pipeline Failed!'
        }
        always {
            // Reports archive karo
            archiveArtifacts \
                artifacts: 'reports/**/*',
                allowEmptyArchive: true

            // Workspace clean karo
            cleanWs()
        }
    }
}

