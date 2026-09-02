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

        //stage('install cluster Operator') {
            //steps {
                    //sh 'curl -L -o cluster-operator.yml https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml'
                    //sh "sed -i 's/rabbitmq-system/rbmq-test/g' cluster-operator.yml"
                    //sh './kubectl get pod -n rbmq-test'
                    //sh './kubectl apply -f cluster-operator.yml -n rbmq-test'
                    // sh 'kubectl apply -f https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml -n rbmq-test'
            //}  
        //}

        stage('deploy cluster RabbitMQ') {
            steps {
                    sh './kubectl apply -f rabbitmq-trove-cluster.yaml -n rbmq-test'
            }
        }

        stage('attendi pod') {
            steps {
                sh './kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=trove-rabbitmq-test -n rbmq-test --timeout=300s'
            }
        }

        stage('create vhosts and users') {
            steps {
                withCredentials([
                        credentialsId: 'trove_roma' , variable: 'trove_roma'
                        credentialsId: 'trove_milano' , variable: 'trove_milano'
                ]) {
                sh '''
                    ./kubectl get pod -n rbmq-test
                    ./kubectl exec -n rbmq-test trove-rabbitmq-test-server-0 -- \
                        rabbitmqctl add_vhost trove-roma || true
                    ./kubectl exec -n rbmq-test trove-rabbitmq-test-server-0 -- \
                        rabbitmqctl add_user trove-roma "$trove_roma" || true
                    ./kubectl exec -n rbmq-test trove-rabbitmq-test-server-0 -- \
                        rabbitmqctl set_permissions -p trove-roma trove-roma ".*" ".*" ".*"


                    ./kubectl exec -n rbmq-test trove-rabbitmq-test-server-0 -- \
                        rabbitmqctl add_vhost trove-milano || true
                    ./kubectl exec -n rbmq-test trove-rabbitmq-test-server-0 -- \
                        rabbitmqctl add_user trove-milano "$trove_milano" || true
                    ./kubectl exec -n rbmq-test trove-rabbitmq-test-server-0 -- \
                        rabbitmqctl set_permissions -p trove-milano trove-milano ".*" ".*" ".*"
                '''
                }
            }
        }

        stage('check') {
            steps {
                sh '''
                    echo "RabbitMQ Cluster"
                    ./kubectl get rabbitmqcluster trove-rabbitmq-test -n rbmq-test
                    echo "Pod RabbitMq"
                    ./kubectl get pod -n rbmq-test -l app.kubernetes.io/name=trove-rabbitmq-test
                    echo "Vhost"
                    ./kubectl exec -n rbmq-test trove-rabbitmq-test-server-0 -- rabbitmqctl list_vhosts
                    echo "Utenti"
                    ./kubectl exec -n rbmq-test trove-rabbitmq-test-server-0 -- rabbitmqctl list_users
                    echo "Permissions Roma"
                    ./kubectl exec -n rbmq-test trove-rabbitmq-test-server-0 -- rabbitmqctl list_permissions -p trove-roma
                    echo "Permissions Milano"
                    ./kubectl exec -n rbmq-test trove-rabbitmq-test-server-0 -- rabbitmqctl list_permissions -p trove-milano
                '''
            }
        }
    }
}

