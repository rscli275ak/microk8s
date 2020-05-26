# MicroK8S - Installation et configuration

## Liens utiles
https://www.youtube.com/watch?v=OTBzaU1-thg

## Configuration faite sur une Ubuntu Server 20.04 LTS

* VMware Personal Edition :
   - 2 CPU, 2 Go de RAM
   - Réseau de type pont (les métriques grafana ne passent pas en NAT)

* VirtualBox :
    - [TODO]

* Configuration :
   - Activer microk8s à l'installation
   - Activer le serveur OpenSSH à l'installation

### Se connecter depuis un shell

    ssh [IP_VM]

### Donner les droits microk8s à l'utilisateur actuel
    sudo usermod -aG microk8s ${USER}
    sudo chown -f -R ${USER} ~/.kube

Se reconnecter ensuite.

### Activer les fonctionnalités standard

    microk8s.enable dashboard dns

Ceci a pour but de déployer grafana entre autres. Pour accéder au dashboard depuis un navigateur, il faut un token d'authentification.

### Récupérer le jeton d'authenfification :

Entrer les commandes suivantes :

    token=$(microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
    microk8s kubectl -n kube-system describe secret $token

On récupère alors le jeton affiché par la variable *token*, que l'on enregistre

### Exposer les services

* Première méthode :

Modifier le service souhaité en remplaçant le type de service de **ClusterIP** à **NodePort** :

    # Please edit the object below. Lines beginning with a '#' will be ignored,
    # and an empty file will abort the edit. If an error occurs while saving this file will be
    # reopened with the relevant failures.
    #
    apiVersion: v1
    kind: Service
    metadata:
    creationTimestamp: 2018-05-08T15:03:48Z
    labels:
    k8s-app: kubernetes-dashboard
    name: kubernetes-dashboard
    namespace: kube-system
    resourceVersion: "1855185"
    selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard
    uid: 02c97f8b-52d1-11e8-a941-080027efcddc
    spec:
    clusterIP: 10.107.194.2xx
    externalTrafficPolicy: Cluster
    ports:
    - nodePort: 32414
    port: 443
    protocol: TCP
    targetPort: 8443
    selector:
    k8s-app: kubernetes-dashboard
    sessionAffinity: None
    type: NodePort  ### clusterIP to NodePort
    status:
    loadBalancer: {}

* Deuxième méthode

Mettre en place un proxy :

Proxy pour accéder au tableau de bord

    microk8s.kubectl proxy --address=10.0.2.15

Configurer la redirection de port dans l'hyperviseur :

    127.0.0.1:2200 <-> 10.0.2.15:22
    172.0.0.1:8000 <-> 10.0.0.15:8001 # ou port proxy

Il faudra spécifier l'accès par proxy dans chaque url :

    http://localhost:8000/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

### Accéder au dashboard

En entrant la commande suivante, on obtient le port externe exposé :

    microk8s.kubectl -n kube-system get services
    kubernetes-dashboard        NodePort    10.152.183.126   <none>        443:32191/TCP            43m

Accéder à l'interface en entrant l'IP de la VM suivie du port exposé en HTTPS.

Coller alors le jeton d'authentification préalablement copié.

### Accéder aux autres services depuis l'extérieur (Grafana)

    microk8s.kubectl cluster-info

Fournit une liste d'URLS pour chaque service :

    Kubernetes master is running at https://127.0.0.1:16443
    Heapster is running at https://127.0.0.1:16443/api/v1/namespaces/kube-system/services/heapster/proxy
    CoreDNS is running at https://127.0.0.1:16443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    Grafana is running at https://127.0.0.1:16443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
    InfluxDB is running at https://127.0.0.1:16443/api/v1/namespaces/kube-system/services/monitoring-influxdb:http/proxy

Pour accéder au service exposé il faut remplacer 127.0.0.1 par l'IP de la VM.

A la demande des identifiants, il faut récupérer le mot de passe par la commande suivante :

    microk8s.config

Grafana est un moniteur de performances sur les différents niveaux de kubernetes (clusters /nodes / pods / conteneurs).

### Commandes de base

Afficher l'ensemble des namespaces :

    microk8s.kubectl get all --all-namespaces

Afficher l'ensemble des services :

    microk8s.kubectl -n kube-system get services

## Démo microbot

Démonstration de scaling sur plusieurs instances d'un même site.

### Création du déploiement

    microk8s.kubectl create deployment microbot --image=dontrebootme/microbot:v1

### Pour vérifier les instances

    microk8s.kubectl get all

Les instances sont listées aux niveau des pods (*pod/*)

### Suppression du déploiement

    microk8s.kubectl delete deployment microbot

### Nombre de répliques souhaitées

    microk8s.kubectl scale deployment microbot --replicas=2

### Supprimer un pod

    microk8s.kubectl delete pod/[POD_NAME]

La suppression d'un pod regénère systématiquement un nouveau pod, en adéquation avec les règles de scalabilité données.

### Exposer un service

    microk8s.kubectl expose deployment microbot --type=NodePort --port=80 --name=microbot-service

Le port exposé s'obtient en entrant la commande suivanet et en recherchant le service exposé :

    microk8s.kubectl get all --all-namespaces

    default       service/microbot-service            NodePort    10.152.183.149   <none>        80:31003/TCP

### Supprimer un service

    microk8s.kubectl delete service microbot.service