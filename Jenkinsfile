// Jenkins CI/CD pipeline for the movie-service and cast-service applications.
//
// - Builds and tests the Docker images.
// - Pushes them to Docker Hub.
// - Deploys them to the dev, qa, staging, and prod Kubernetes namespaces.
//
// Production deployment is allowed only from master and requires manual approval.

pipeline {
    // Run the pipeline on any available Jenkins agent.
    agent any

    environment {
        // Docker Hub namespace used for both application images.
        DOCKER_ID = "seryonya"

        // Tag images with the Jenkins build ID (e.g. v.5).
        DOCKER_TAG = "v.${BUILD_ID}"

        // Load the Kubernetes configuration stored in Jenkins credentials under the ID "config".
        KUBECONFIG = credentials('config')
    }

    stages {
        // Build Docker images for both application services.
        stage('Build') {
            steps {
                sh '''
                    docker build -t $DOCKER_ID/movie-service:$DOCKER_TAG ./movie-service
                    docker build -t $DOCKER_ID/cast-service:$DOCKER_TAG ./cast-service
                '''
            }
        }

        // Start the complete Docker Compose stack and verify that both service endpoints respond.
        stage('Test') {
            steps {
                sh '''
                    docker compose down -v || true
                    docker compose up -d movie_db cast_db
                    timeout 30 sh -c 'until docker compose exec -T movie_db pg_isready -U movie_db_username -d movie_db_dev; do sleep 1; done'
                    timeout 30 sh -c 'until docker compose exec -T cast_db pg_isready -U cast_db_username -d cast_db_dev; do sleep 1; done'
                    docker compose up -d --build movie_service cast_service nginx
                    sleep 3
                    curl -f http://localhost:8080/api/v1/casts/docs > /dev/null
                    curl -f http://localhost:8080/api/v1/movies/docs > /dev/null
                    docker compose down -v
                '''
            }
        }

        // Authenticate with Docker Hub and push both versioned images.
        stage('Push') {
            environment {
                // Load the Docker Hub access token from Jenkins credentials.
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

        // Deploy both services to the dev namespace using Helm.
        stage('Deploy dev') {
            steps {
                sh '''
                    helm upgrade --install cast charts -f charts/values-cast.yaml --set image.tag=$DOCKER_TAG --namespace dev
                    helm upgrade --install movie charts -f charts/values-movie.yaml --set image.tag=$DOCKER_TAG --namespace dev
                '''
            }
        }

        // Deploy both services to the qa namespace using Helm.
        stage('Deploy qa') {
            steps {
                sh '''
                    helm upgrade --install cast charts -f charts/values-cast.yaml --set image.tag=$DOCKER_TAG --namespace qa
                    helm upgrade --install movie charts -f charts/values-movie.yaml --set image.tag=$DOCKER_TAG --namespace qa
                '''
            }
        }

        // Deploy both services to the staging namespace using Helm.
        stage('Deploy staging') {
            steps {
                sh '''
                    helm upgrade --install cast charts -f charts/values-cast.yaml --set image.tag=$DOCKER_TAG --namespace staging
                    helm upgrade --install movie charts -f charts/values-movie.yaml --set image.tag=$DOCKER_TAG --namespace staging
                '''
            }
        }

        // Deploy to production only from master and only after manual approval.
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