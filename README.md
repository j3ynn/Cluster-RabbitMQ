# Documentazione - RabbitMQ Cluster Operator

Pipeline Jenkins per automatizzare il deployment di un cluster RabbitMQ su Kubernetes.

## Obiettivo Pipeline

La pipeline automatizza l'intero processo di installazione
di RabbitMQ all'interno del cluster Kubernetes.

## Cosa esegue

Automatizza i seguenti passaggi:

- Scarica `kubectl` nel workspace del build
- Installa il Cluster Operator di RabbitMQ
- Fa il deploy di RabbitMQ applicando il manifest custom `rabbitmq-trove-cluster.yaml`
- Crea vhost e utenti dedicati

---

## Autenticazione al cluster

Le credenziali del cluster sono salvate su Jenkins come Credential di tipo *Secret file* e passate alla pipeline tramite la variabile d'ambiente `KUBECONFIG`:

```groovy
environment {
    KUBECONFIG = credentials('kubeconfig-bellucci')
}
```

`kubectl` legge questa variabile per sapere a quale cluster collegarsi, come autenticarsi e quale contesto usare.

---

## Stage - download kubectl

Questo stage scarica il programma Kubectl
poichè Jenkins possa comunicare con il cluster kubernetesed ed eseguire i comandi 

- Tramite `curl` scarico il programma `kubectl` nel workspace del build
- Cambia i permessi con `chmod +x`, rendendo il file eseguibile


```groovy
sh '''
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    chmod +x kubectl
'''
```

## Stage - install cluster Operator

In questo stage viene installato il RabbitMQ Cluster Operator,
necessario per permettere a Kubernetes di creare e gestire il cluster RabbitMQ.

- Tramite `curl` scarica il manifest ufficiale dell'Operator RabbitMQ e lo salva come `cluster-operator.yml`
- Con `sed` sostituisce ogni occorrenza del namespace `rabbitmq-system` con `rbmq-test`, il namespace con la quale lavoro
- Applica il manifest modificato con `kubectl apply`

```groovy
sh 'curl -L -o cluster-operator.yml https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml'
sh "sed -i 's/rabbitmq-system/rbmq-test/g' cluster-operator.yml"
sh './kubectl apply -f cluster-operator.yml -n rbmq-test'
```

## Stage - deploy cluster RabbitMQ

In questo stage viene creato il cluster RabbitMQ all'interno di Kubernetes
utilizzando il manifest custom rabbitmq-trove-cluster.yaml

- Applica il manifest custom `rabbitmq-trove-cluster.yaml`, che descrive il cluster RabbitMQ
(numero di repliche, immagine, risorse CPU/memoria, storage persistente, configurazione)

```groovy
sh './kubectl apply -f rabbitmq-trove-cluster.yaml -n rbmq-test'
```

## Stage - create vhosts and users

Crea le utenze dedicate dentro RabbitMQ, una per sede (Roma, Milano)

- Le password sono salvate come Credential Jenkins separate per ogni sede (`trove_roma`, `trove_milano`), così da non scriverle mai in chiaro nel repo
- Vengono recuperate come variabili tramite `withCredentials`, da usare all'interno dello stage
- Si crea il vhost dedicato alla sede
- Si crea l'utente con la password recuperata dalla Credential
- Si assegnano all'utente permessi completi solo sul proprio vhost, isolandolo dagli altri

```groovy
withCredentials([
    string(credentialsId: 'trove_roma', variable: 'trove_roma')
]) {
    sh '''
        ./kubectl exec -n rbmq-test trove-rabbitmq-server-0 -- \
            rabbitmqctl add_vhost trove-roma
        ./kubectl exec -n rbmq-test trove-rabbitmq-server-0 -- \
            rabbitmqctl add_user trove-roma "$trove_roma"
        ./kubectl exec -n rbmq-test trove-rabbitmq-server-0 -- \
            rabbitmqctl set_permissions -p trove-roma trove-roma ".*" ".*" ".*"
    '''
}
```
