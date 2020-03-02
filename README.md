# Velero 101
[Velero](https://velero.io/) is an open source tool to safely backup and restore, perform disaster recovery, and migrate Kubernetes cluster resources and persistent volumes.
Features:
* Back up Clusters - Backup your Kubernetes resources and volumes for an entire cluster, or part of a cluster by using namespaces or label selectors.

* Schedule Backups - Set schedules to automatically kickoff backups at recurring intervals.

* Backup Hooks - Configure pre and post-backup hooks to perform custom operations before and after Velero backups.

In this workshop we will setup end to end 101 demo of Velero.
All you need a Kubernetes context in place and `kubectl`.

## MinIO setup
For this demo we will use MinIO for backup storage.
MinIO is a high performance distributed object storage server, designed for large-scale private cloud infrastructure. MinIO is designed in a cloud-native manner to scale sustainably in multi-tenant environments. Orchestration platforms like Kubernetes provide perfect cloud-native environment to deploy and scale MinIO.
?"Refer to [MinIO Deployment on Kubernetes](https://docs.min.io/docs/deploy-minio-on-kubernetes.html) documentation.
For this demo we will setup Minio using following commands.

----
kubectl create -f https://github.com/minio/minio/blob/master/docs/orchestration/kubernetes/minio-standalone-pvc.yaml?raw=true

kubectl create -f https://github.com/minio/minio/blob/master/docs/orchestration/kubernetes/minio-standalone-deployment.yaml?raw=true

kubectl create -f https://github.com/minio/minio/blob/master/docs/orchestration/kubernetes/minio-standalone-service.yaml?raw=true
----

This should setup necessary resources to run MinIO locally in your Kubernetes cluster.
The MinIO service is of type LoadBalancer, which means you need to be on public cloud using their native Kubernetes distribution or
Tanzu PKS.
Run the following commands to get status of the service.

----
kubectl get endpoints

kubectl get svc
----

You should see a service named *minio-service*.
You can access the MinIO using http://[external-ip]:9000, credentials (minio/minio123).

## Setup Velero
We will use Velero v1.2.0 for this demo.
You can find Velero installation docs [here](https://velero.io/docs/v1.2.0/basic-install/).
I've a Mac so I just executed `brew install velero`.

Next we will setup Velero server on the Kubernetes cluster you've access to.
1. Create a `credentials-velero` file add MinIO credentials to it.
----
[default]

aws_access_key_id = minio

aws_secret_access_key = minio123
----

2. Login to MinIO console (`http://[external-ip]:9000`) and create a bucket called *velero*.

3. Run following to start the Velero in your Kubernetes cluster.

----
velero install \
--provider aws \
--plugins velero/velero-plugin-for-aws:v1.0.0 \
--bucket velero \
--secret-file ./credentials-velero \
--use-volume-snapshots=false \
--backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://[external-ip]:9000
----

Successful deployment should show following messages

`Velero is installed! ⛵ Use 'kubectl logs deployment/velero -n velero' to view the status.`

Run `kubectl get all -l component=velero --namespace=velero` to confirm.

This completes Velero setup for the demo.

## Setup NGINX and prepare the backup
1. Deploy NGINX using sample YAML given by Velero.

`kubectl apply -f https://raw.githubusercontent.com/vmware-tanzu/velero/master/examples/nginx-app/base.yaml`

2. Confirm the deployment

`kubectl get deployments --namespace=nginx-example`

3. Start the Veleo backup

`velero backup create nginx-backup --selector app=nginx`

4. Check how the backup is created on MinIO using `http://[external-ip]:9000/minio/velero/`

5. Check status of the backup

`velero describe backups nginx-backup`

You should see `Phase:  Completed`.

## Recover the using the backup
1. Once the backup is complete we can simulate a disaster by deleting the namespace

`kubectl delete namespace nginx-example`

Confirm that the app and all the resources are deleted.

`kubectl get all --namespace=nginx-example`

2. Start the restore process

`velero restore create --from-backup nginx-backup`

3. Check the status of the restore

`velero restore get`

Get details of the restore name

`velero restore describe [name of the restore]`

4. Confirm that the app has been restored

`kubectl get ns` should show `nginx-example`.

`kubectl get all -n nginx-example`

5. You can also check the restore has created separate folders on MinIO `http://[external-ip]:9000/minio/velero/restores/`

## Clean up
1. If you want to delete any backups you created, including data in object storage and persistent volume snapshots, you can run

----
velero backup get

velero backup delete nginx-backup
----

2. Verify that backup and restore of `nginx-backup` is removed.

----
velero backup get

velero restore get
----

Also, check in MinIO console that the folders are removed `http://[external-ip]:9000/minio/velero/`.
