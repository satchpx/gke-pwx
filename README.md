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

```
PX_POD=$(kubectl get pods -l name=portworx -n kube-system \
    -o jsonpath='{.items[0].metadata.name}')

alias pxctl="kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl"
```

## Install Postgres

```
kubectl apply -f specs/postgres-px.yaml
```

```
kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
postgres-5b6898557d-7tcr8   1/1     Running   0          12s

```

```
pxctl volume inspect `kubectl get pvc | grep postgres-data | awk '{print $3}'`
Volume	:  924591656571108482
	Name            	 :  pvc-7434340e-5932-11ea-892e-42010a8e01ae
	Size            	 :  5.0 GiB
	Format          	 :  ext4
	HA              	 :  3
	IO Priority     	 :  HIGH
	Creation time   	 :  Feb 27 07:26:22 UTC 2020
	Shared          	 :  no
	Status          	 :  up
	State           	 :  Attached: ecaa1e08-9a7e-402b-b289-795f8c31e750 (10.142.15.204)
	Device Path     	 :  /dev/pxd/pxd924591656571108482
	Labels          	 :  namespace=default,pvc=postgres-data,repl=3,io_profile=db
	Reads           	 :  37
	Reads MS        	 :  36
	Bytes Read      	 :  376832
	Writes          	 :  1581
	Writes MS       	 :  7376
	Bytes Written   	 :  133541888
	IOs in progress 	 :  0
	Bytes used      	 :  31 MiB
	Replica sets on nodes:
		Set 0
		  Node 		 : 10.142.0.3 (Pool 01527bd7-49dc-403d-8326-14c85c0b0741 )
		  Node 		 : 10.142.0.2 (Pool eb6e5ba5-5a7a-40dd-a815-a2a3bbcaa6ba )
		  Node 		 : 10.142.15.204 (Pool 8ada593d-a47a-4119-bc07-6d0d4f2067ce )
	Replication Status	 :  Up
	Volume consumers	 :
		- Name           : postgres-5b6898557d-7tcr8 (75215ae1-5932-11ea-bb1a-42010a8e01ad) (Pod)
		  Namespace      : default
		  Running on     : gke-amex-demo-east-default-pool-1720da34-0m0f
		  Controlled by  : postgres-5b6898557d (ReplicaSet)
```

### Benchmark

```
POD=`kubectl get pods -l app=postgres | grep Running | grep 1/1 | awk '{print $1}'`
kubectl exec -it $POD bash
root@postgres-95c4d46d5-4sqb9:/#su postgres
```

Next, run the read test.
```
root@postgres-95c4d46d5-4sqb9:/# pgbench -i -s 70 postgres & pgbench -c 4 -j 2 -T 600 -S postgres
```

Run the write test.
```
root@postgres-95c4d46d5-4sqb9:/# pgbench -i -s 70 postgres & pgbench -c 4 -j 2 -T 600 -N postgres
```

Now, run the random write test. 
```
root@postgres-95c4d46d5-4sqb9:/# pgbench -i -s 70 postgres & pgbench -i -s 70 postgres;
```

### App consistent snapshots
```
kubectl exec -it $POD bash
root@postgres-5b6898557d-7tcr8:/# su postgres
$ psql
psql (10.1)
Type "help" for help.

postgres-# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)

```

#### Create Database
```
kubectl exec -it $POD bash
root@postgres-5b6898557d-7tcr8:/# su postgres
$ createdb pxdemo
$
$
$ psql
psql (10.1)
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 pxdemo    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)
```

#### Write some data
```
postgres=# \q
could not save history to file "/home/postgres/.psql_history": No such file or directory
$ pgbench -i -s 50 pxdemo
```

Verify
```
$ psql -a pxdemo -c 'select count(*) from pgbench_accounts;'
select count(*) from pgbench_accounts;
  count
---------
 5000000
(1 row)
```

#### Create object store credentials
```
pxctl credentials create --provider s3 \
--s3-access-key <redacted> \
--s3-secret-key <redacted> \
--s3-region us-east-1 \
--s3-endpoint <redacted> \
--s3-disable-ssl \
minio
```

#### Apply pre-snap rule
```
kubectl apply -f postgres-3dsnap-prerule.yaml
rule.stork.libopenstorage.org/postgres-3dsnap-prerule created
```

```
kubectl get rule
NAME                      AGE
postgres-3dsnap-prerule   5s
```

#### Create 3d-cloudsnap
```
kubectl apply -f cloudsnap-3d-postgres.yaml
volumesnapshot.volumesnapshot.external-storage.k8s.io/postgres-3d-cloudsnapshot created
```

```
kubectl  get volumesnapshots
NAME                        AGE
postgres-3d-cloudsnapshot   9s
```

```
pxctl cs status
NAME					SOURCEVOLUME		STATE		NODE		TIME-ELAPSED	COMPLETED
63c23c14-5936-11ea-892e-42010a8e01ae	924591656571108482	Backup-Done	10.142.15.204	15.321343646s	Thu, 27 Feb 2020 07:54:49 UTC
```

```
pxctl cs list
SOURCEVOLUME						SOURCEVOLUMEID			CLOUD-SNAP-ID					CREATED-TIME				TYPE			STATUS
pvc-7434340e-5932-11ea-892e-42010a8e01ae		924591656571108482		47f6ade9-94a8-4654-aac1-e06bd6f47ce5/924591656571108482-314230151652709092		Thu, 27 Feb 2020 07:54:33 UTC		StorkManual		Done
```

## Delete the cluster
After the experiment is complete, destroy:
```
gcloud container clusters delete demo-east
gcloud container clusters delete demo-west
```