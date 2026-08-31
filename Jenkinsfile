pipeline {
    agent any

    environment {
        KUBECONFIG = credentials('kubeconfig-bellucci')
    }

    stages {

        stage('scarica kubectl') {
            steps {
                sh '''
                    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                    chmod +x kubectl
                '''
            }
        }

        stage('install cluster Operator') {
            steps {
                    sh 'curl -L -o cluster-operator.yml https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml'
                    sh "sed -i 's/rabbitmq-system/rbmq-test/g' cluster-operator.yml"
                    sh './kubectl get pod -n rbmq-test'
                    //sh './kubectl apply -f cluster-operator.yml'
                    //sh 'kubectl apply -f https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml -n rbmq-test'
                }  
            }
    }

}
