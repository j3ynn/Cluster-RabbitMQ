# Documentazione - RabbitMQ Cluster

Pipeline Jenkins per automatizzare il deployment di un cluster RabbitMQ su Kubernetes.

## Obiettivo Pipeline

La pipeline automatizza l'intero processo di installazione e confugurazione
di RabbitMQ all'interno del cluster Kubernetes.

## Cosa esegue

Automatizza i seguenti passaggi:

- Scarica `kubectl` nel workspace del build
- Fa il deploy di RabbitMQ applicando il manifest custom `rabbitmq-trove-cluster.yaml`
- Crea vhost e utenti dedicati

*pipeline idempotente*

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

## Stage - Check

Verifica finale che mostra, in un unico output, lo stato di tutto quello che la pipeline ha creato:

- Il cluster RabbitMQ (`RabbitmqCluster`)
- I pod RabbitMQ
- I vhost, sia di Roma che di Milano
- Gli utenti, sia di Roma che di Milano
- I permessi assegnati a ciascun utente sul proprio vhost


## Note - perché non ho installato l'Operator

RabbitMQ Cluster Operator è installato a livello di cluster,
osservare le risorse RabbitmqCluster creando e gestendo automaticamente le risorse necessarie per il funzionamento di RabbitMQ.
L'Operator viene installato una sola volta e può essere utilizzato per gestire cluster RabbitMQ presenti in diversi namespace.
La sua installazione comprende anche risorse condivise a livello di cluster, come la CRD rabbitmqclusters.rabbitmq.com, ClusterRole, ClusterRoleBinding e webhook.
Nel cluster esiste già un'installazione dell'Operator utilizzata da altri ambienti.
- Per questo la pipeline non reinstalla né modifica l'Operator: una nuova applicazione del manifest ufficiale potrebbe modificare componenti dell'installazione esistente, ad esempio versione, immagine o webhook, con possibili conseguenze sugli altri cluster RabbitMQ.
La pipeline utilizza quindi l'Operator già installato e si limita a creare la risorsa RabbitmqCluster nel namespace di test tramite il manifest rabbitmq-trove-cluster.yaml.

---
