pipeline {
environment { 
DOCKER_ID = "fa6060" 
DOCKER_IMAGE_MOVIE = "jenkins_devops_exams_movie_service"
DOCKER_IMAGE_CAST = "jenkins_devops_exams_cast_service"
DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
DOCKER_PASS = credentials("DOCKER_HUB_PASS") 
}


agent any // Jenkins will be able to select all available agents
stages {
        stage(' Docker Build'){ // docker build image stage
            steps {
                script {
                sh '''
                 docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG cast-service/
                 docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG cast-service/
                sleep 6
                '''
                }
            }
        }
        stage('Docker run'){ // run container from our builded image
                steps {
                    script {
                    sh '''
                    docker run -d -p 8000:8000 --name movie-service $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                    docker run -d -p 8001:8001 --name cast-service $DOCKER_ID/$DOCKER_IMAGE_CASR:$DOCKER_TAG
                    sleep 10
                    '''
                    }
                }
            }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                    curl -f http://localhost:8000/ || exit 1
                    curl -f http://localhost:8001/ || exit 1
                    '''
                    }
            }

        }
        stage('Docker Push'){ //we pass the built image to our docker hub account
           
            steps {

                script {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                '''
                }
            }

        }

stage('Deploiement en dev'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/movie-service/values.yaml values-movie.yml
                cat values-movie.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-movie.yml

                cp fastapi/cast-service/values.yaml values-cast.yml
                cat values-cast.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-cast.yml
                
                helm upgrade --install movie-service fastapi/movie-service --values=values-movie.yml --namespace dev
                helm upgrade --install cast-service fastapi/cast-service --values=values-cast.yml --namespace dev
                '''
                }
            }

        }
stage('Deploiement en staging'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/movie-service/values.yaml values-movie.yml
                cat values-movie.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-movie.yml

                cp fastapi/cast-service/values.yaml values-cast.yml
                cat values-cast.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-cast.yml
                
                helm upgrade --install movie-service fastapi/movie-service --values=values-movie.yml --namespace staging
                helm upgrade --install cast-service fastapi/cast-service --values=values-cast.yml --namespace staging
                '''
                }
            }

        }
  stage('Deploiement en prod'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
            // Create an Approval Button with a timeout of 15minutes.
            // this require a manuel validation in order to deploy on production environment
                    timeout(time: 15, unit: "MINUTES") {
                        input message: 'Do you want to deploy in production ?', ok: 'Yes'
                    }

                script {
                sh '''
                 rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/movie-service/values.yaml values-movie.yml
                cat values-movie.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-movie.yml

                cp fastapi/cast-service/values.yaml values-cast.yml
                cat values-cast.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-cast.yml
                
                helm upgrade --install movie-service fastapi/movie-service --values=values-movie.yml --namespace prod
                helm upgrade --install cast-service fastapi/cast-service --values=values-cast.yml --namespace prod
                '''
                }
            }

        }


}
}