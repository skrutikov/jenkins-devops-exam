pipeline {
    agent any

    environment {
        DOCKER_ID = "seryonya"
        MOVIE_IMAGE = "movie-service"
        CAST_IMAGE = "cast-service"
        DOCKER_TAG = "v.${BUILD_ID}"
    }

    stages {
        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG ./movie-service
                    docker build -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG ./cast-service
                '''
            }
        }

        stage('Docker Compose Test') {
            steps {
                sh '''
                    docker compose down -v || true
                    docker compose up -d --build
                    sleep 10
                    curl -f http://localhost:8080/api/v1/casts/docs > /dev/null
                    curl -f http://localhost:8080/api/v1/movies/docs > /dev/null
                '''
            }
            post {
                always {
                    sh 'docker compose down -v || true'
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials('DOCKER_HUB_PASS')
            }
            steps {
                sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin
                    docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
                    docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
                '''
            }
            post {
                always {
                    sh 'docker logout || true'
                }
            }
        }

        stage('Deploy dev') {
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                sh '''
                    helm upgrade --install cast charts -f charts/values-cast.yaml --set image.tag=$DOCKER_TAG --namespace dev --create-namespace --wait --timeout 2m
                    helm upgrade --install movie charts -f charts/values-movie.yaml --set image.tag=$DOCKER_TAG --namespace dev --create-namespace --wait --timeout 2m
                '''
            }
        }

        stage('Deploy qa') {
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                sh '''
                    helm upgrade --install cast charts -f charts/values-cast.yaml --set image.tag=$DOCKER_TAG --namespace qa --create-namespace --wait --timeout 2m
                    helm upgrade --install movie charts -f charts/values-movie.yaml --set image.tag=$DOCKER_TAG --namespace qa --create-namespace --wait --timeout 2m
                '''
            }
        }

        stage('Deploy staging') {
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                sh '''
                    helm upgrade --install cast charts -f charts/values-cast.yaml --set image.tag=$DOCKER_TAG --namespace staging --create-namespace --wait --timeout 2m
                    helm upgrade --install movie charts -f charts/values-movie.yaml --set image.tag=$DOCKER_TAG --namespace staging --create-namespace --wait --timeout 2m
                '''
            }
        }

        stage('Deploy prod') {
            when {
                expression {
                    env.GIT_BRANCH == 'origin/master' || env.GIT_BRANCH == 'master'
                }
            }
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Deploy to production?', ok: 'Yes'
                }

                sh '''
                    helm upgrade --install cast charts -f charts/values-cast.yaml --set image.tag=$DOCKER_TAG --namespace prod --create-namespace --wait --timeout 2m
                    helm upgrade --install movie charts -f charts/values-movie.yaml --set image.tag=$DOCKER_TAG --namespace prod --create-namespace --wait --timeout 2m
                '''
            }
        }
    }
}
