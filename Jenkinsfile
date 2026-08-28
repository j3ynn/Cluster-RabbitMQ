pipeline {
    agent any

    environment {
        KUBECONFIG = credentials('tim-tcos-kubeconfig')
    }

    stages {

        stage('install cluster Operator') {
            steps {
                sh 'kubectl apply -f https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml -n rbmq-test'
            }
        }

        stage('create namespace applicativo') {
            steps {
                sh 'kubectl create namespace trove-messaging'
            }
        }

        stage('deploy cluster RabbitMQ') {
            steps {
                sh 'kubectl apply -f rabbitmq-trove-cluster.yaml'
            }
        }

        stage('create vhosts and users') {
            steps {
                sh '''
                    kubectl exec -n trove-messaging trove-rabbitmq-server-0 -- \
                        rabbitmqctl add_vhost trove-roma
                    kubectl exec -n trove-messaging trove-rabbitmq-server-0 -- \
                        rabbitmqctl add_user trove-roma <PASSWORD-SICURA>
                    kubectl exec -n trove-messaging trove-rabbitmq-server-0 -- \
                        rabbitmqctl set_permissions -p trove-roma trove-roma ".*" ".*" ".*"
                '''
            }
        }
    }
}