# Projet d'un cluster Kubernetes avec Kubeadm et Microk8s sur environnement Ubuntu/Linux ---- Setp by step

  # Conditions préalables
  - Une paire de clés SSH sur votre machine Linux / macOS / BSD locale.
  - Trois nodes ( hôtes ) exécutant Ubuntu 18.04 ou latest avec 2 Go de RAM et 2 vCPU chacun en LABO-INSPQ
  - Ansible installé sur votre node master ( elle servira de Maitre pour les 2 workers )
  - Installation de docker pour le lancement d'un / plusieurs conteneur a partir des images sur chaque node
  - Installation de kubelet et kubeadm ( snap install kubelet --classic  ,  snapt install kubeadm --classic )sur chaque hôte
  - Installer microk8s sur chaque machine ( snap install microk8s --classic) sur chaque hôte
  - Installer kubectl sur chaque hôte ( snap install kubectl --classic)
  
  # Étape 1 - Création de nodes et ajout de nodes au cluster avec kubeadm
   
   # Initialisation de kubeadm sur la machine master
      kubeadm init
      
   # Ajout de nodes au cluster
      kubeadm join ''coller le token initialisé depuis le master node''
   
   # Vérifier votre installation avec la commande
      kubectl get nodes
  
  # Étape 2 - Lancement d'un POD ( nous lancerons l'application wordpress avec une base de donnée MYSQL )
      kubectl create -f wordpres_pod.yaml
      
   Vous verrez que notre POD des paramètres additionnel comme le scheduling qui permet d'ajouter des contraintes de deploiement de la ligne 6 à 14 en d'autre termes sélectionne    le node sur lequel un Pod sera déployé dans notre casnous l'avons déployé sur un node dont le label disktype a la valeur ssd. 
      
   -- *Vérfication que le POD a bien été crée       
      
      kubectl get po db     // db ici le nom du POD   
   
   -- *Vérification du status du POD
      
      kubect get po/db 
   
   # Nous pouvons ajouter des labels sur l'un des nodes du cluster
  
      kubectl label nodes NODE_NAME disktype=ssd
   
 # Étape 3 - Mise en place de Service tel que ClusterIP et le NodePort
  
  Le clusterIP : Permet d'exposer un ensemble de POD sur le cluster en d'autres termes permet d'autres PODs du cluster pourront y accéder via un namespace
  Le NodePort : Permet d'exposer les PODs à l'extérieur du cluster en définissant un numéro de port sur les machines du cluster.
  
  # Mise en place d'un ClusterIP
   
   -- *Création du POD NGINX avec une image alpine 
      
      kubectl create -f nginx_pod.yaml
 
   -- *Définition du Service de type ClusterIp
     
      kubect create -f clusterIp.yaml
   
   -- *Accès au Service depuis le cluster : nous allons créer un POD debug avec une image alpine juste pour le test
      
      kubectl create -f debug_pod.yaml
      
   -- *Visualiser la resources services en tapant la commande:
      
      kubectl get servicees web
   
   -- *Pour avoir la spécificaton d'Uun Service / POD / Deployment au format yaml exécutez la commande:
      
      kubectl get svc/'nom du service' -o yaml
      kubectl get po/'nom du pod' -o yaml
      kubectl get deploy/'nom du deploy' -o yaml
   -- *Pour avoir les détails d'Uun service/ POD/ Deployment éxcutez la commande :
   
      kubectl describe svc/'nom du service'
      kubectl describe po/'nom du POD'
      kubectl describe deploy/'nom du deploy'
   -- *Pour supprimer un service/ POD/ Deployment
      
      kubectl delete svc/'nom du service'
      kubectl delete po/'nom du service'
      kubectl detele deploy/ 'nom du deploy'
      
  # Mise en place d'un nodePort 
  
      kubectl create -f nodeport.yaml
  
 # Étape 4 - Déploiement d'un application et l'exposer à l'exterieur du cluster via un service de type NodePort en utilisant la webapp instant/vote
 
      kubectl create -f vote_deplyment.yaml
   
   -- Expostion dues Pods du Deployment
   
      kubectl create -f vote_service.yaml
      
   -- Vous trouveverez une bonne de mise à jour d'un Deployment ici:
      
      https://kubernetes.io/fr/docs/concepts/workloads/controllers/deployment/
   
   -- Mise à jour d'Un replicat : Vous verrez dans le fichier vote_deployment.yaml le replicat déjà effectuer de la ligne 10 à 15 et des ressources utiles pour le replicat à           partir de la ligne 26 
    
 # Étape 5 - Installation du Metrics Server : utilie pour mettre en place le metrics-server en charge de récuperer les metrics(CPU, memoire) de consommations des Pods
   
   -- il est nécessaire de déployer le process metrics-server avec la commande suivante:
   
      kubectl apply -f https://github.com/kubernetes-sigs/metricsserver/releases/download/v0.3.6/components.yaml
   
   Après un ntemps de latence de 10s effectuez la commande:
   
      kubectl top nodes
    
   Nous sommes prêt à installer la ressource HorizontalPodAutoscaler permettant de modifier le nombre de réplicas du Deployment si celui-ci utilise plus de 10% du CPU qui lui      est alloué
   
        kubectl apply -f hpa.yaml
   
   Vérifiez que l'HorizontalPodAutoscaler a été créé correctement:
   
      kubectl get hpa
    
   # -- *Test
   Pour envoyer un grand nombre de requête sur le service, nous allons utiliser l'outils Apache Bench.
    Utilisez la commande suivante en remplaçant NODE_IP par l'adresse IP de l'un des nodes de
    votre cluster (vous pouvez obtenir les IPs des nodes à l'aide de $ kubectl get nodes -o wide ):
    Note: assurez vous d'avoir installer Docker sur votre machine au préalable
    Revenez à votre espace de travail et créez un playbook nommé workers.yml:
  
      $ docker run lucj/ab -n 100000 -c 50 http://NODE_IP:30100/
  
   Depuis un autre terminal, observez l'évolution de la consommation du CPU et l'augmentation du nombre de réplicas (cela peux prendre quelques minutes)
    
    kubectl get -w hpa
    
    Note: l'option -w (watch) met à jour régulièrement le résultat de la commande.
 
 # Étape 6 - Création de namespaces

   *Les commandes suivantes permettent de créer les namespaces < development et production >
  
    $ kubectl create namespace development
    $ kubectl create namespace production
    
   *Lister les namespaces
  
    $ kubectl get namespace
    
    
   # -- *Création de Deployment avec namespaces ( dans notre cas on deploie le serveur nginx basé sur alpine su les 3 namespaces )
   
    $ kubectl run web-0 --image nginx:1.14-alpine    // deploiement sur le namespace default qui prend l'etiquétage 0 
    $ kubectl run web-1 --image nginx:1.14-alpine --namespace development    // deploiement sur le namespace development qui prend l'etiquétage 1 
    $ kubectl run www-2 --image nginx:1.14-alpine --namespace production    // deploiement sur le namespace production qui pprend l'étiquetage 2
  
   -- *Lister l'ensemble des Deployments et Pods

    $ kubectl get deploy,po --all-namespaces
   
   -- *Supprimer un ou plusieurs namespaces
   
    $ kubectl delete ns/development ns/production
    
   # -- *Limitation des ressources dans un namespace
   
   Celui-ci défini une ressource de type ResourceQuota qui limitera l'utilisation de la mémoire et
   du cpu dans le namespace associé. Au sein de celui-ci:
    *chaque container devra spécifier des demandes et des limites pour la RAM et le cpu
    *l'ensemble des containers ne pourra pas demander plus de 1GB de RAM
    *l'ensemble des containers ne pourra pas utiliser plus de 2GB de RAM
    *l'ensemble des containers ne pourra pas demander plus d'1 cpu
    *l'ensemble des containers ne pourra pas utiliser plus de 2 cpus
 
   En utilisant la commande suivante, créez cette nouvelle ressource en l'associant au namespace development.
   
     $ kubectl apply -f quota.yaml --namespace=development
    
   --*Container de ce Pod définit des demandes et des limites pour la RAM et le CPU ( création d'un pod ngnix qui définit les demandes et des limites RAM, CPU )
    
     $ kubectl apply -f pod-quota-1.yaml
    
   --*Vérifiez que le Pod a été créé correctement:
  
     $ kubectl get po -n development
     
   --*Vérification de l'utilisation du quota
    
   Utilisez la commande suivante pour voir les ressources RAM et CPU utilisée au sein du namespace:

     $ kubectl get resourcequota quota --namespace=development --output=yaml
     
   # *NB: Pour supprimer un QUOTA il suffit de supprimer un namespace
  
