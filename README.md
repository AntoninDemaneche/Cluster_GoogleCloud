# DEVOPS - Cluster Google cloud
Thomas Dumont
Antonin Demaneche

## I - Configuration : 

### A. - Création du cluster : 

On crée un cluster sur google cloud : 

**![](https://image.prntscr.com/image/971yYjqlSNaTNun-B6wYwg.png)**
Une fois chose faite, il faut se rendre dans Kubernetes Engine et Selectionner Cluster.

**![](https://image.prntscr.com/image/uaN4s6lYQ8Kl7Zm3J050xA.png)**

Vous pouvez ensuite crée le cluster en créant la configuration que vous souhaitez.

**![](https://image.prntscr.com/image/hthAHgnmRGKn3UDSNN2F8w.png)**

Une fois chose il s'agira de configuré la connection au cluster.
### B. - Connection au Cluster : 

On install le SDK Google, pour crée une connection de notre pc à au cluster.
```
https://cloud.google.com/sdk/auth_success
```
Apres installation du SKA se rendre sur kubernetes engine > cluster > se connecter pour obtenir la commande permettant de se conencter au cluster depuis le cli

Pour tester kubectl get node : 

```
NAME                                       STATUS   ROLES    AGE     VERSION
gke-cluster-1-default-pool-54885cfb-6897   Ready    <none>   4h45m   v1.16.15-gke.4901
gke-cluster-1-default-pool-54885cfb-ggkj   Ready    <none>   4h45m   v1.16.15-gke.4901
gke-cluster-1-default-pool-54885cfb-lv5r   Ready    <none>   4h45m   v1.16.15-gke.4901
```

## II - Persitant Volume Claim 

* Deployez le fichier de manifeste : 

```
kubectl apply -f $WORKING_DIR/wordpress-volumeclaim.yaml
```
* Pour verifié l'état du PVC : 
```
akhadimer@Machinator3000:~$ kubectl get persistentvolumeclaim
NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
wordpress-volumeclaim   Bound    pvc-2d2e2fde-aec2-4c48-a40a-bc4b29caea14   200Gi      RWO            standard       115m
```

* Contenu du fichier afin de créer le volume :
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: wordpress-volumeclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
```

## III - Base de données

* Crée une instance mysql : 
```
INSTANCE_NAME=mysql-wordpress-instance
gcloud sql instances create $INSTANCE_NAME
```
* Crée un utilisateur à la base de donnée : 
```
CLOUD_SQL_PASSWORD=$(openssl rand -base64 18)
gcloud sql users create wordpress --host=% --instance $INSTANCE_NAME \
    --password $CLOUD_SQL_PASSWORD
```

## IV - Wordpress 
### A. - Configuration Wordpress

* Ajouter le rôle cloudsql.client au compte :
```
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --role roles/cloudsql.client \
    --member serviceAccount:$SA_EMAIL
```

* Créer une clé pour le compte de service :
```
gcloud iam service-accounts keys create $key.json \
    --iam-account $SA_EMAIL
```

* Créer un identifiant pour MySQL :
```
kubectl create secret generic cloudsql-db-credentials \
    --from-literal username=wordpress \
    --from-literal password=$CLOUD_SQL_PASSWORD
```

* Créer un secret pour les IDs du compte de service :
`kubectl create secret generic cloudsql-instance-credentials \
    --from-file key.json`



### B. - Déploiement de Wordpress

Déploier le [fichier](https://github.com/AntoninDemaneche/Cluster_GoogleCloud/blob/main/wordpress_cloudsql.yaml) manifeste :
`kubectl create -f wordpress_cloudsql.yaml`

Voir si le déploiement a bien été fait : 
```
akhadimer@Machinator3000:~$ kubectl get pod -l app=wordpress --watch
NAME                         READY   STATUS    RESTARTS   AGE
wordpress-789975cc54-mmpq5   2/2     Running   0          94m
```


### C. - Résultats 


Pour trouver l'ip du wordpress ( Prendre External Ip ) : 

```
akhadimer@Machinator3000:~$kubectl get svc -l app=wordpress --watch
NAME        TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
wordpress   LoadBalancer   10.4.1.14    35.233.62.158   80:31349/TCP   43m
```

Le site wordpress fonctionne correctement :

**![](https://image.prntscr.com/image/jLr0S8iwQJS367ToVk_WLQ.png)**
