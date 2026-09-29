pipeline {
    agent any

    environment {
        DOCKER_ID = "seryonya"
        DOCKER_TAG = "v.${BUILD_ID}"
        KUBECONFIG = credentials('config')
    }

    stages {
        stage('Build') {
            steps {
                sh '''
                    docker build -t $DOCKER_ID/movie-service:$DOCKER_TAG ./movie-service
                    docker build -t $DOCKER_ID/cast-service:$DOCKER_TAG ./cast-service
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    docker compose down -v || true
                    docker compose up -d --build
                    sleep 10
                    curl -f http://localhost:8080/api/v1/casts/docs > /dev/null
                    curl -f http://localhost:8080/api/v1/movies/docs > /dev/null
                    docker compose down -v
                '''
            }
        }

        stage('Push') {
            environment {
                DOCKER_PASS = credentials('DOCKER_HUB_PASS')
            }
            steps {
                sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin
                    docker push $DOCKER_ID/movie-service:$DOCKER_TAG
                    docker push $DOCKER_ID/cast-service:$DOCKER_TAG
                '''
            }
        }

        stage('Deploy dev') {
            steps {
                sh '''
                    helm upgrade --install cast charts -f charts/values-cast.yaml --set image.tag=$DOCKER_TAG --namespace dev
                    helm upgrade --install movie charts -f charts/values-movie.yaml --set image.tag=$DOCKER_TAG --namespace dev
                '''
            }
        }

        stage('Deploy qa') {
            steps {
                sh '''
                    helm upgrade --install cast charts -f charts/values-cast.yaml --set image.tag=$DOCKER_TAG --namespace qa
                    helm upgrade --install movie charts -f charts/values-movie.yaml --set image.tag=$DOCKER_TAG --namespace qa
                '''
            }
        }

        stage('Deploy staging') {
            steps {
                sh '''
                    helm upgrade --install cast charts -f charts/values-cast.yaml --set image.tag=$DOCKER_TAG --namespace staging
                    helm upgrade --install movie charts -f charts/values-movie.yaml --set image.tag=$DOCKER_TAG --namespace staging
                '''
            }
        }

        stage('Deploy prod') {
            when {
                expression { env.GIT_BRANCH == 'origin/master' }
            }
            steps {
                input message: 'Deploy to production?', ok: 'Yes'
                sh '''
                    helm upgrade --install cast charts -f charts/values-cast.yaml --set image.tag=$DOCKER_TAG --namespace prod
                    helm upgrade --install movie charts -f charts/values-movie.yaml --set image.tag=$DOCKER_TAG --namespace prod
                '''
            }
        }
    }
}
