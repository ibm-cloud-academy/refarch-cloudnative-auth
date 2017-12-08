podTemplate(
    label: 'mypod',
    volumes: [
      hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
      secretVolume(secretName: 'registry-account', mountPath: '/var/run/secrets/registry-account'),
      configMapVolume(configMapName: 'registry-config', mountPath: '/var/run/configs/registry-config')
    ],

    containers: [
        containerTemplate(name: 'java', image: 'openjdk:8-jdk-alpine', alwaysPullImage: true, ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'docker' , image: 'docker:17.06.1-ce', ttyEnabled: true, command: 'cat')
    ],
) {
    node ('mypod') {
        checkout scm
        container('java') {
            stage('Build Gradle Project') {
                sh """
                #!/bin/bash
                ./gradlew build -x test
                ./gradlew docker
                """
            }
        }
        container('docker') {
            stage ('Build Docker Image') {
                sh """
                    #!/bin/bash
                    NAMESPACE=`cat /var/run/configs/registry-config/namespace`
                    REGISTRY=`cat /var/run/configs/registry-config/registry`
                    docker build -t \${REGISTRY}/\${NAMESPACE}/bluecompute-auth:${env.BUILD_NUMBER} .                    
                """
            }
            stage ('Push Docker Image to Registry') {
                sh """
                #!/bin/bash
                NAMESPACE=`cat /var/run/configs/registry-config/namespace`
                REGISTRY=`cat /var/run/configs/registry-config/registry`

                set +x
                DOCKER_USER=`cat /var/run/secrets/registry-account/username`
                DOCKER_PASSWORD=`cat /var/run/secrets/registry-account/password`
                docker login -u=\${DOCKER_USER} -p=\${DOCKER_PASSWORD} \${REGISTRY}
                set -x

                docker push \${REGISTRY}/\${NAMESPACE}/bluecompute-auth:${env.BUILD_NUMBER}
                """
            }
        }
        container('kubectl') {
            stage('Deploy new Docker Image') {
                sh """
                #!/bin/bash
                set +e
                NAMESPACE=`cat /var/run/configs/registry-config/namespace`
                REGISTRY=`cat /var/run/configs/registry-config/registry`
                DEPLOYMENT=`kubectl get deployments -l app=bluecompute,micro=auth -o name`

                kubectl get deployments \${DEPLOYMENT}

                if [ \${?} -ne "0" ]; then
                    # No deployment to update
                    echo 'No deployment to update'
                    exit 1
                fi

                # Update Deployment
                kubectl set image deployment/\${DEPLOYMENT} web=\${REGISTRY}/\${NAMESPACE}/bluecompute-auth:${env.BUILD_NUMBER}
                kubectl rollout status deployment/\${DEPLOYMENT}
                """
            }
        }
    }
}
