pipeline {
    agent any

    environment {
        KUBECONFIG = credentials('kubeconfig-bellucci')
    }

    stages {

        stage('download kubectl') {
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
                    sh './kubectl apply -f cluster-operator.yml -n rbmq-test'
                    //sh 'kubectl apply -f https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml -n rbmq-test'
                }  
            }

        stage('deploy cluster RabbitMQ') {
            steps {
                    sh './kubectl apply -f rabbitmq-trove-cluster.yaml -n rbmq-test'
            }
        }

        stage('attendi pod') {
            steps {
                sh './kubectl wait --for=condition=Ready pod --all -n rbmq-test --timeout=300s'
            }
        }

        stage('create vhosts and users') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'trove_roma' ,
                        variable: 'trove_roma'
                    )
                ]) {
                sh '''
                    ./kubectl get pod -n rbmq-test
                    ./kubectl exec -n rbmq-test trove-rabbitmq-server-0 -- \
                        rabbitmqctl add_vhost trove-roma
                    ./kubectl exec -n rbmq-test trove-rabbitmq-server-0 -- \
                        rabbitmqctl add_user trove-roma "$trove_roma"
                    ./kubectl exec -n rbmq-test trove-rabbitmq-server-0 -- \
                        rabbitmqctl set_permissions -p trove-roma trove-roma ".*" ".*" ".*"
                '''
                }
            }
        }
    }
}
