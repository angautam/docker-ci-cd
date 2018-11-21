  env.DOCKERHUB_USERNAME = 'angautam'

  node("docker-prod") {
    stage("Production") {
      try {
        // Create the service if it doesn't exist otherwise just update the image
        sh '''
          SERVICES=$(docker service ls --filter name=docker-ci-cd --quiet | wc -l)
          if [[ "$SERVICES" -eq 0 ]]; then
            docker network rm docker-ci-cd || true
            docker network create --driver overlay --attachable docker-ci-cd
            docker service create --replicas 3 --network docker-ci-cd --name docker-ci-cd -p 9098:8080 ${DOCKERHUB_USERNAME}/docker-ci-cd:${BUILD_NUMBER}
          else
            docker service update --image ${DOCKERHUB_USERNAME}/docker-ci-cd:${BUILD_NUMBER} docker-ci-cd
          fi
          '''
        // run some final tests in production
        checkout scm
        sh '''
          sleep 60s 
          for i in `seq 1 20`;
          do
            STATUS=$(docker service inspect --format '{{ .UpdateStatus.State }}' docker-ci-cd)
            if [[ "$STATUS" != "updating" ]]; then
              docker run --rm -v ${WORKSPACE}:/go/src/docker-ci-cd --network docker-ci-cd -e SERVER=docker-ci-cd golang go test docker-ci-cd -v --run Integration
              break
            fi
            sleep 10s
          done
          
        '''
      }catch(e) {
        sh "docker service update --rollback  docker-ci-cd"
        error "Service update failed in production"
      }finally {
        sh "docker ps -aq | xargs docker rm || true"
      }
    }
  }
