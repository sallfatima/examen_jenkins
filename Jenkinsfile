pipeline {
    environment { 
        DOCKER_ID = "fa6060"
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
        KUBECONFIG = credentials("config")
        DOCKER_TAG = "v.${BUILD_ID}.0"
        DOCKER_IMAGE_MOVIE = "jenkins_devops_exams_movie_service"
        DOCKER_IMAGE_CAST = "jenkins_devops_exams_cast_service"
        NETWORK_NAME = "my_network"
    }

    agent any


 

    stages {
        stage('Cleanup Previous Containers') {
            steps {
                script {
                   echo "🛑 Arrêt et suppression des anciens conteneurs..."
                    sh '''
                    
                        docker ps -aq | xargs -r docker stop || true
                        docker ps -aq | xargs -r docker rm || true
                   
                    '''
                }
            }
        }

        stage('Setup Docker Network') {
            steps {
                script {
                    echo "🔗 Vérification du réseau Docker..."
                    sh '''
                    docker network ls | grep ${NETWORK_NAME} || docker network create ${NETWORK_NAME}
                    '''
                }
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    sh '''
                    echo "🚀 Construction des images Docker..."
                    docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG movie-service/
                    docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG cast-service/

                    echo "📂 Liste des images Docker disponibles :"
                    docker images
                    '''
                }
            }
        }

        stage('Docker Run') {
            steps {
                script {
                   
                    sh '''
                    echo "🚀 Démarrage des services..."
                    docker run -d --network=my_network --name movie-db -e POSTGRES_USER=movie_db_username -e POSTGRES_PASSWORD=movie_db_password -e POSTGRES_DB=movie_db_dev postgres:15 || echo "⚠️ Erreur lors du démarrage de movie-db"
                    docker run -d --network=my_network --name cast-db -e POSTGRES_USER=cast_db_username -e POSTGRES_PASSWORD=cast_db_password -e POSTGRES_DB=cast_db_dev postgres:15 || echo "⚠️ Erreur lors du démarrage de cast-db"
                    
                    echo "🕒 Attente du démarrage des bases de données..."
                    sleep 10

                    docker run -d --network=my_network -p 80:80 --name nginx nginx:latest || echo "⚠️ Erreur lors du démarrage de nginx"
                    docker run -d --network=my_network -p 32000:8000 --name movie-service $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG || echo "⚠️ Erreur lors du démarrage de movie-service"
                    docker run -d --network=my_network -p 32010:8000 --name cast-service $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG || echo "⚠️ Erreur lors du démarrage de cast-service"

                    echo "🔍 Logs du service movie-service..."
                    docker logs movie-service || true

                    echo "🔍 Logs du service cast-service..."
                    docker logs cast-service || true

                  
    
                    echo "📂 Vérification des conteneurs en cours d'exécution..."
                    docker ps -a

                   


                    '''
                }
            }
        }

        stage('Diagnostic') {
            steps {
                script {
                    echo "🛠 Diagnostic des services en cours..."

                    // Vérifier les logs de movie-service
                    sh '''
                    echo "🔍 Logs de movie-service :"
                    docker logs movie-service || echo "⚠️ Impossible de récupérer les logs"
                    '''

                    // Vérifier si l'application écoute bien sur le port 8000
                    sh '''
                    echo "🔎 Vérification des ports ouverts dans movie-service..."
                    docker exec movie-service sh -c 'netstat -tulnp' || echo "⚠️ Erreur lors de la récupération des ports"
                    '''

                    // Tester l'accès à l'API depuis l'intérieur du conteneur
                    sh '''
                    echo "🌐 Test d'accès à l'API depuis movie-service..."
                    docker exec movie-service curl -s http://localhost:8000 || echo "⚠️ API inaccessible à l'intérieur du conteneur"
                    '''

                    // Vérifier la connexion à la base de données
                    sh '''
                    echo "🗄 Vérification de la connexion entre movie-service et movie-db..."
                    docker exec movie-service sh -c 'nc -zv movie-db 5432' || echo "⚠️ Problème de connexion à la base de données"
                    '''
                }
            }
        }


        stage('Test Acceptance') {
            steps {
                 script {
                    echo "🔎 Vérification de l'état des services..."

                    sh '''
                    if curl -s http://localhost:32000 | grep "Welcome"; then
                        echo "✅ movie-service est accessible !"
                    else
                        echo "❌ movie-service inaccessible !" && exit 1
                    fi

                    if curl -s http://localhost:32010 | grep "Welcome"; then
                        echo "✅ cast-service est accessible !"
                    else
                        echo "❌ cast-service inaccessible !" && exit 1
                    fi
                    '''
                    }
            }
        }
        
        stage('Docker Push') {
            steps {
                script {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin

                    echo "📤 Pushing images to Docker Hub..."
                    docker push $DOCKER_ID/jenkins_devops_exams_movie_service:$DOCKER_TAG
                    docker push $DOCKER_ID/jenkins_devops_exams_cast_service:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Déploiement en Dev') {
            steps {
                script {
                    sh '''
                    echo "🚀 Déploiement sur Kubernetes (namespace: dev)..."

                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config

                    cp fastapi/movie-service/values.yaml values-movie.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-movie.yml

                    cp fastapi/cast-service/values.yaml values-cast.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-cast.yml
                    
                    cp fastapi/movie-db/values.yaml values-movie-db.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-movie-db.yml

                    cp fastapi/cast-db/values.yaml values-cast-db.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-cast-db.yml

                    cp fastapi/nginx/values.yaml values-nginx.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-nginx.yml

                    helm upgrade --install movie-service fastapi/movie-service --values=values-movie.yml --namespace dev
                    helm upgrade --install cast-service fastapi/cast-service --values=values-cast.yml --namespace dev
                    helm upgrade --install movie-db fastapi/movie-db --values=values-movie-db.yml --namespace dev
                    helm upgrade --install cast-db fastapi/cast-db --values=values-cast-db.yml --namespace dev
                    helm upgrade --install nginx fastapi/nginx --values=values-nginx.yml --namespace dev

                    '''
                }
            }
        }
         stage('Déploiement en QA') {
            steps {
                script {
                    sh '''
                    echo "🚀 Déploiement sur Kubernetes (namespace: staging)..."

                    cp fastapi/movie-service/values.yaml values-movie.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-movie.yml

                    cp fastapi/cast-service/values.yaml values-cast.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-cast.yml
                    
                    cp fastapi/movie-db/values.yaml values-movie-db.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-movie-db.yml

                    cp fastapi/cast-db/values.yaml values-cast-db.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-cast-db.yml

                    cp fastapi/nginx/values.yaml values-nginx.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-nginx.yml

                    helm upgrade --install movie-service fastapi/movie-service --values=values-movie.yml --namespace qa
                    helm upgrade --install cast-service fastapi/cast-service --values=values-cast.yml --namespace qa
                    helm upgrade --install movie-db fastapi/movie-db --values=values-movie-db.yml --namespace qa
                    helm upgrade --install cast-db fastapi/cast-db --values=values-cast-db.yml --namespace qa
                    helm upgrade --install nginx fastapi/nginx --values=values-nginx.yml --namespace qa

                    '''
                }
            }
        }
        stage('Déploiement en Staging') {
            steps {
                script {
                    sh '''
                    echo "🚀 Déploiement sur Kubernetes (namespace: staging)..."

                    cp fastapi/movie-service/values.yaml values-movie.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-movie.yml

                    cp fastapi/cast-service/values.yaml values-cast.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-cast.yml
                    
                    cp fastapi/movie-db/values.yaml values-movie-db.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-movie-db.yml

                    cp fastapi/cast-db/values.yaml values-cast-db.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-cast-db.yml

                    cp fastapi/nginx/values.yaml values-nginx.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-nginx.yml

                    helm upgrade --install movie-service fastapi/movie-service --values=values-movie.yml --namespace staging
                    helm upgrade --install cast-service fastapi/cast-service --values=values-cast.yml --namespace staging
                    helm upgrade --install movie-db fastapi/movie-db --values=values-movie-db.yml --namespace staging
                    helm upgrade --install cast-db fastapi/cast-db --values=values-cast-db.yml --namespace staging
                    helm upgrade --install nginx fastapi/nginx --values=values-nginx.yml --namespace staging

                    '''
                }
            }
        }

        stage('Déploiement en Prod') {
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: '✅ Valider le déploiement en production ?'
                }
                script {
                    sh '''
                    echo "🚀 Déploiement sur Kubernetes (namespace: prod)..."

                     cp fastapi/movie-service/values.yaml values-movie.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-movie.yml

                    cp fastapi/cast-service/values.yaml values-cast.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-cast.yml
                    
                    cp fastapi/movie-db/values.yaml values-movie-db.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-movie-db.yml

                    cp fastapi/cast-db/values.yaml values-cast-db.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-cast-db.yml

                    cp fastapi/nginx/values.yaml values-nginx.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-nginx.yml

                    helm upgrade --install movie-service fastapi/movie-service --values=values-movie.yml --namespace prod
                    helm upgrade --install cast-service fastapi/cast-service --values=values-cast.yml --namespace prod
                    helm upgrade --install movie-db fastapi/movie-db --values=values-movie-db.yml --namespace prod
                    helm upgrade --install cast-db fastapi/cast-db --values=values-cast-db.yml --namespace prod
                    helm upgrade --install nginx fastapi/nginx --values=values-nginx.yml --namespace prod

                    '''
                }
            }
        }
    }
}