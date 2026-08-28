pipeline {
    agent any

    stages {

        stage('install cluster Operator') {
            steps {
                withKubeConfig(credentialsId: 'tim-tcos-kubeconfig') {
                    sh 'curl -L -o cluster-operator.yml https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml'
                    sh "sed -i 's/rabbitmq-system/rbmq-test/g' cluster-operator.yml"
                    sh 'kubectl apply -f cluster-operator.yml'
                     //sh 'kubectl apply -f https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml -n rbmq-test'
                }  
            }
        }

    }
}