podTemplate(
    label: 'gradlePod',
    volumes: [
      hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
      secretVolume(secretName: 'icpadmin', mountPath: '/var/run/secrets/registry-account'),
      configMapVolume(configMapName: 'icpconfig', mountPath: '/var/run/configs/registry-config')
    ],

    containers: [
        containerTemplate(name: 'gradle', image: 'ibmcase/gradle:jdk8-alpine', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'kubectl', image: 'ibmcloudacademy/k8s-icp:v1.0', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'docker' , image: 'docker:17.06.1-ce', ttyEnabled: true, command: 'cat')
    ],
) {
    node ('gradlePod') {
        checkout scm
        container('gradle') {
            stage('Build Gradle Project') {
                sh """
                #!/bin/sh
                gradle build -x test
                gradle docker
                """
            }
        }
        container('docker') {
            stage ('Build Docker Image') {
                sh """
                    #!/bin/bash
                    NAMESPACE=`cat /var/run/configs/registry-config/namespace`
                    REGISTRY=`cat /var/run/configs/registry-config/registry`

                    cd docker
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
                DOCKER_USER=`cat /var/run/secrets/registry-account/username`
                DOCKER_PASSWORD=`cat /var/run/secrets/registry-account/password`

                curl -kLo cloudctl https://10.10.1.10:8443/api/cli/cloudctl-linux-amd64
                curl -kLo helm.tgz https://10.10.1.10:8443/api/cli/helm-linux-amd64.tar.gz
                tar -xzf helm.tgz
                ./linux-amd64/helm init --client-only
                chmod a+x cloudctl
                ./cloudctl login -a https://10.10.1.10:8443 -u \${DOCKER_USER} -p \${DOCKER_PASSWORD} -n default

                ./linux-amd64/helm repo add bluecompute https://raw.githubusercontent.com/ibm-cloud-academy/icp-jenkins-helm-bluecompute/master/charts
                ./linux-amd64/helm install --tls -n bluecompute-auth --set image.repository=\${REGISTRY}/\${NAMESPACE}/bluecompute-auth --set image.tag=${env.BUILD_NUMBER} --set customer.service.url="http://bluecompute-customer-customer:8080/" bluecompute/auth

                """
            }
        }
    }
}
