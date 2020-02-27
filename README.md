# gke-pwx

## Install gke clusters
```
gcloud container clusters create demo-east \
     --region us-east1 \
     --node-locations us-east1-b,us-east1-c,us-east1-d \
     --disk-type=pd-ssd \
     --disk-size=50GB \
     --labels=portworx=gke \
     --machine-type=n1-standard-4 \
     --num-nodes=1 \
     --image-type ubuntu \
     --scopes compute-rw,storage-ro \
     --enable-autoscaling --max-nodes=6 --min-nodes=3
```

... and the second cluster
```
gcloud container clusters create demo-west \
     --region us-west1 \
     --node-locations us-west1-b,us-west1-c,us-west1-a \
     --disk-type=pd-ssd \
     --disk-size=50GB \
     --labels=portworx=gke \
     --machine-type=n1-standard-4 \
     --num-nodes=1 \
     --image-type ubuntu \
     --scopes compute-rw,storage-ro \
     --enable-autoscaling --max-nodes=6 --min-nodes=3
```

## Install Portworx

```
gcloud services enable compute.googleapis.com
```

### Provide permissions to Portworx
These next steps grant Kubernetes ClusterRole level permissions to your Gcloud account. These permissions then allow you to install Portworx. 

Run the following to find your Gcloud account name. 
```
gcloud info --format='value(config.account)'
```

Using the output of the above as the ACCOUNT_NAME, run the command below to provide ClusterRole permissions to your Gcloud account.
```
kubectl create clusterrolebinding cluster-admin-binding \
            --clusterrole=cluster-admin \
            --user=ACCOUNT_NAME
```

```
kubectl apply -f specs/px.yaml
```

## Install Postgres

```
kubectl apply -f specs/postgres-px.yaml
```

## Delete the cluster
After the experiment is complete, destroy:
```
gcloud container clusters delete demo-east
gcloud container clusters delete demo-west
```