# Documentazione - RabbitMQ Cluster local

Pipeline Jenkins per automatizzare il deployment di un cluster RabbitMQ su Kubernetes.

## Obiettivo Pipeline

La pipeline automatizza l'intero processo di installazione e confugurazione
di RabbitMQ all'interno del cluster Kubernetes locale.

## Cosa esegue

Automatizza i seguenti passaggi:

- Scarica `kubectl` nel workspace del build
- installa `cert-menager`
- applica il manifest per l'installazione del cluster Operator
- creo namespace `cluster-rabbitmq` e il deploy di RabbitMQ applicando il manifest custom `rabbitmq-trove-cluster.yaml`
- Crea vhost e utenti dedicati

*pipeline idempotente*

---

## Autenticazione al cluster

Le credenziali del cluster sono salvate su Jenkins come Credential di tipo *Secret file* e passate alla pipeline tramite la variabile d'ambiente `KUBECONFIG`:

```groovy
environment {
    KUBECONFIG = credentials('kubeconfig-cluster-rabbitmq')
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

## Stage - install cert-manager

Questo stage installa il tool cert-menager che automatizza la gestione dei certificati TLS, per la cifratura dei dati e autenticità.
*un Operator ha bisdogno di cert-Manager per poter funzionare, perchè non sa come gestire i certificati TLS* 

## Stage - cluster Operator

Applica il manifest precedentemente scaricato con la giusta versione, per installare il cluster Operator

```kubectl apply -f https://github.com/rabbitmq/cluster-operator/releases/download/v2.16.1/cluster-operator.yml
```

## Stage - deploy cluster RabbitMQ

In questo stage viene creato il cluster RabbitMQ all'interno di Kubernetes
utilizzando il manifest custom rabbitmq-trove-cluster.yaml

- crea il namespace dedicato `cluster-rabbitmq`
- Applica il manifest custom `rabbitmq-trove-cluster.yaml`, che descrive il cluster RabbitMQ
(numero di repliche, immagine, risorse CPU/memoria, storage persistente, configurazione)

```groovy
sh './kubectl apply -f rabbitmq-trove-cluster.yaml -n rbmq-test'
```

## Stage - create vhosts and users

Crea le utenze dedicate dentro RabbitMQ (Roma, Milano)

- Le password sono salvate come Credential Jenkins separate(`trove_roma`, `trove_milano`), così da non scriverle mai in chiaro nel repo
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

## Stage - Check

Verifica finale che mostra, in un unico output, lo stato di tutto quello che la pipeline ha creato:

- I namespace
- I vhost, sia di Roma che di Milano
- Gli utenti, sia di Roma che di Milano
- I permessi assegnati a ciascun utente sul proprio vhost

---
