# Backup OpenEBS cStor by Velero (for Windows Powershell user)

## Objectives

Ensure to backup and restore OpenEBS cStor based volume snapshot with AWS S3 bucket.

> ⚠️ This document is written for Windows Powershell users.
You only need Windows to run it. because you may not have the WSL environment.

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Objectives](#objectives)
- [Prerequisites](#prerequisites)
- [Setup Velero for OpenEBS cStor](#setup-velero-for-openebs-cstor)
  - [Value definitions explanation](#value-definitions-explanation)
- [Install](#install)
- [Install Velero tool](#install-velero-tool)
- [Create AWS S3 bucket for bucket access](#create-aws-s3-bucket-for-bucket-access)
- [Deploy Velero](#deploy-velero)
  - [Create a secret file for AWS S3 bucket access](#create-a-secret-file-for-aws-s3-bucket-access)
  - [Deploy Velero](#deploy-velero-1)
    - [Add OpenEBS Velero plugin](#add-openebs-velero-plugin)
- [Deploy example application](#deploy-example-application)
  - [Deploy](#deploy)
    - [Access example application for recording access.log via port forwarding.](#access-example-application-for-recording-accesslog-via-port-forwarding)
- [Backup](#backup)
  - [Check details of the backup](#check-details-of-the-backup)
- [Delete example application](#delete-example-application)
- [Restore](#restore)
  - [Check backup](#check-backup)
  - [Restore](#restore-1)
  - [Check details of the restore](#check-details-of-the-restore)
  - [Set iSCSI target IP address to Pool Pod](#set-iscsi-target-ip-address-to-pool-pod)
    - [Get volume name](#get-volume-name)
    - [Get PVC service cluster IP address](#get-pvc-service-cluster-ip-address)
      - [PVC service cluster IP address](#pvc-service-cluster-ip-address)
    - [Get Pool Pod name](#get-pool-pod-name)
    - [Modify iSCSI target IP in Pool Pod](#modify-iscsi-target-ip-in-pool-pod)
      - [Modify targetip to $pv_ip pool/$volume_name](#modify-targetip-to-pv_ip-poolvolume_name)
- [Check restore](#check-restore)
- [Retrieve access.log](#retrieve-accesslog)
- [Delete example application](#delete-example-application-1)

<!-- /code_chunk_output -->

## Prerequisites
<details><summary>Prerequisites</summary>

- OpenEBS installed
  - See [OpenEBS cStor install for Free IBM Cloud (for Windows PowerShell user)](https://github.com/ymikasa/openebs-cstor-install-to-free-ibm-cloud)

- Created cStor Storage Class: openebs-sparse-sc
  - See [OpenEBS cStor install for Free IBM Cloud (for Windows PowerShell user)](https://github.com/ymikasa/openebs-cstor-install-to-free-ibm-cloud)

- The AWS account ID/Access Key environment variable is set for bucket creation.


```powershell
$env:AWS_ACCOUNT_ID=020000000000 <# AWS Account ID for creating unique bucket name (optional) #>
$env:AWS_ACCESS_KEY_ID=AKIA...
$env:AWS_SECRET_ACCESS_KEY=...
```

- scoop installed

</details>


## Setup Velero for OpenEBS cStor

### Value definitions explanation

<details><summary>Values for tools</summary>

| Key  |  Sample value | Description |
| - | - |- |
| $home | | User home folder |
| $env:AWS_ACCOUNT_ID | 020000000000 | AWS Account ID for creating unique bucket name (optional) |
| $env:AWS_ACCESS_KEY_ID | AKIA****** | |
| $env:AWS_SECRET_ACCESS_KEY | ****** | |
| $home/.kube | | Kubernetes configuration folder |
| $bucket | velero-${env:AWS_ACCOUNT_ID} |
| $region | us-east-2 |
| $job | PowerShell background job of port-foward |
| $pv_ip | 172.21.152.42 | PVC iSCSI target IP |
| $volume_name | pvc-e9a4ed7c-aa68-440a-88cf-fc799b3781a9 | PVC volume name |

</details>

<details><summary>Files</summary>

| File  | Description |
| - | - |
| $home/.kube/credentials-velero | AWS S3 credentials for bucket access |

</details>


## Install 

## Install Velero tool

```powershell
scoop install velero
```

## Create AWS S3 bucket for bucket access

```powershell
$bucket="velero-${env:AWS_ACCOUNT_ID}"
$region="us-east-2"
aws s3api create-bucket `
  --bucket $bucket `
  --region $region `
  --create-bucket-configuration LocationConstraint=$region

aws s3api put-bucket-encryption --bucket $bucket `
  --server-side-encryption-configuration '{\"Rules\":[{\"ApplyServerSideEncryptionByDefault\":{\"SSEAlgorithm\":\"AES256\"}}]}'
```

## Deploy Velero

### Create a secret file for AWS S3 bucket access

```powershell
@"
[default]
aws_access_key_id=${env:AWS_ACCESS_KEY_ID}
aws_secret_access_key=${env:AWS_SECRET_ACCESS_KEY}
"@ | Set-Content $home/.kube/credentials-velero
```

### Deploy Velero

```powershell
velero install `
  --provider aws `
  --plugins velero/velero-plugin-for-aws:v1.1.0 `
  --bucket $bucket `
  --backup-location-config region=$regionz `
  --snapshot-location-config region=$region `
  --secret-file $home/.kube/credentials-velero `
  --velero-pod-cpu-request 100m `
  --velero-pod-mem-request 128Mi `
  --velero-pod-cpu-limit 100m `
  --velero-pod-mem-limit 128Mi
kubectl -n velero get po
```
Results:
```text
NAME                    READY STATUS  RESTARTS AGE
velero-54957ccbbf-mw6dx 1/1   Running 0        36s
```

#### Add OpenEBS Velero plugin

```powershell
velero plugin add openebs/velero-plugin:1.9.0
kubectl -n velero get po
```
Results:
```text
NAME                    READY STATUS  RESTARTS AGE
velero-59d85ff4f4-ll46z 1/1   Running 0        16s
```

```powershell
@"
apiVersion: velero.io/v1
kind: VolumeSnapshotLocation
metadata:
  name: default
  namespace: velero
spec:
  provider: openebs.io/cstor-blockstore
  config:
    bucket: $bucket
    prefix: openebs
    provider: aws
    region: $region
    s3Url: s3.amazonaws.com
"@ | Set-Content volumesnapshotlocation.yaml
kubectl apply -f volumesnapshotlocation.yaml
```

## Deploy example application

The cStor storage class 'openebs-sparse-sc' is required for volume mount.

```powershell
kubectl get sc
```
Results:
```text
NAME                        PROVISIONER                                                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
openebs-device              openebs.io/local                                           Delete          WaitForFirstConsumer   false                  18h
openebs-hostpath            openebs.io/local                                           Delete          WaitForFirstConsumer   false                  18h
openebs-jiva-default        openebs.io/provisioner-iscsi                               Delete          Immediate              false                  18h
openebs-snapshot-promoter   volumesnapshot.external-storage.k8s.io/snapshot-promoter   Delete          Immediate              false                  18h
openebs-sparse-sc           openebs.io/provisioner-iscsi                               Delete          Immediate              false                  8h
```

### Deploy

```powershell
kubectl apply -f with-cstor-pv.yaml
kubectl -n nginx-example get po
```
Results:
```text
NAME                              READY STATUS  RESTARTS AGE
nginx-deployment-5ccc99bffb-9btzk 2/2   Running 0        25s
```

#### Access example application for recording access.log via port forwarding.

```powershell
$job=start-job -ScriptBlock { kubectl -n nginx-example port-forward svc/my-nginx --address=0.0.0.0 30080:80 }
curl http://localhost:30080/
curl http://localhost:30080/error
$job | stop-job
```

## Backup

```powershell
velero backup create nginx-backup-pv --selector app=nginx
velero backup get
```
Results:
```text
NAME            STATUS    ERRORS WARNINGS CREATED                       EXPIRES STORAGE LOCATION SELECTOR
nginx-backup-pv Completed 0      0        2020-07-27 21:24:40 -0700 PDT 29d     default          app=nginx
```

### Check details of the backup

```powershell
velero backup describe nginx-backup-pv --details
```
Results:
```yaml
Name: nginx-backup-pv
...
Phase: Completed
Errors:    0
Warnings:  0
Label selector:  app=nginx
Storage Location:  default
...
Resource List:
  apps/v1/ReplicaSet:
    - nginx-example/nginx-deployment-5ccc99bffb
  v1/Endpoints:
    - nginx-example/my-nginx
  v1/Namespace:
    - nginx-example
  v1/PersistentVolume:
    - pvc-2ac1df89-5339-4e9e-85a2-cd1931470fc1
  v1/PersistentVolumeClaim:
    - nginx-example/nginx-logs
  v1/Pod:
    - nginx-example/nginx-deployment-5ccc99bffb-kdqc2
  v1/Service:
    - nginx-example/my-nginx
Velero-Native Snapshots:
  pvc-2ac1df89-5339-4e9e-85a2-cd1931470fc1:
    Snapshot ID:        pvc-2ac1df89-5339-4e9e-85a2-cd1931470fc1-velero-bkp-nginx-backup-pv
    Type:               cstor-snapshot
    Availability Zone:
    IOPS:               <N/A>
```

## Delete example application

```powershell
kubectl delete ns nginx-example
```
Results:
```text
namespace "nginx-example" deleted
```

## Restore

### Check backup

```powershell
velero backup get 
```
Results:
```text
NAME            STATUS    ERRORS WARNINGS CREATED                       EXPIRES STORAGE LOCATION SELECTOR
nginx-backup-pv Completed 0      0        2020-07-27 21:24:40 -0700 PDT 26d     default          app=nginx
```

### Restore

```powershell
velero restore create nginx-restore-pv --from-backup nginx-backup-pv --selector app=nginx
velero restore get
```
Results:
```text
NAME             BACKUP          STATUS    ERRORS WARNINGS CREATED                       SELECTOR
nginx-restore-pv nginx-backup-pv Completed 0      2        2020-07-27 21:30:59 -0700 PDT app=nginx
```

### Check details of the restore

```powershell
velero restore describe nginx-restore-pv --details
```
Results:
```yaml
Name: nginx-restore-pv
...
Phase: Completed
...
Warnings:
  Velero: <none>
  Cluster: persistentvolumes "pvc-2ac1df89-5339-4e9e-85a2-cd1931470fc1" not found
  Namespaces:
    nginx-example:  could not restore, persistentvolumeclaims "nginx-logs" already exists. Warning: the in-cluster version is different than the backed-up version.
Backup: nginx-backup-pv
Namespaces:
  Included: all namespaces found in the backup
  Excluded: <none>
Resources:
  Included: *
  Excluded: nodes, events, events.events.k8s.io, backups.velero.io, restores.velero.io, resticrepositories.velero.io
  Cluster-scoped:  auto
Namespace mappings: <none>
Label selector: app=nginx
Restore PVs: auto
```

The restore job is completed with warnings. It means that the target(iSCSI target) volume is missing.

### Set iSCSI target IP address to Pool Pod

After restore for remote backup is completed, you need to set target-ip for the volume in pool pod. You will see below error with suspending in create contaner status.

```powershell
kubectl -n nginx-example describe po -l app=nginx
```
Results:
```text
Events:
 Type    Reason                 Age                   From                    Message
 ----    ------                 ----                  ----                    -------
 Normal  Scheduled              8m16s                 default-scheduler       Successfully assigned nginx-examplenginx-deployment-5ccc99bffb-kdqc2 to 10.144.183.23
 Normal  SuccessfulAttachVolume 8m16s                 attachdetach-controller AttachVolume.Attach succeeded forvolume "pvc-e9a4ed7c-aa68-440a-88cf-fc799b3781a9"
 Warning FailedMount            3m58s (x2 over 6m13s) kubelet, 10.144.183.23  Unable to attach or mount volumes:unmounted volumes=[nginx-logs], unattached volumes=[default-token-n4fmx nginx-logs]: timed out waiting for the condition
 Warning FailedMount            2m6s (x2 over 6m8s)   kubelet, 10.144.183.23  MountVolume.WaitForAttach failed for volume "pvc-e9a4ed7c-aa68-440a-88cf-fc799b3781a9" : failed to get any path for iscsi disk, last err seen:
iscsi: failed to attach disk: Error: iscsiadm: Could not login to [iface: default, target: iqn.2016-09.com.openebs.cstor:pvc-e9a4ed7c-aa68-440a-88cf-fc799b3781a9, portal: 172.21.152.42,3260].
```

#### Get volume name

Get volume name 
Find a volume name by PVC name "nginx-logs".

```
$volume_name=kubectl -n nginx-example get pvc nginx-logs -o jsonpath="{.spec.volumeName}" 
```

#### Get PVC service cluster IP address

```powershell
kubectl -n openebs get svc
```
Results:
```text
NAME                                     TYPE      CLUSTER-IP    EXTERNAL-IP PORT(S)                             AGE
admission-server-svc                     ClusterIP 172.21.212.65 <none>      443/TCP                             10h
maya-apiserver-service                   ClusterIP 172.21.86.180 <none>      5656/TCP                            10h
pvc-e9a4ed7c-aa68-440a-88cf-fc799b3781a9 ClusterIP 172.21.152.42 <none>      3260/TCP,7777/TCP,6060/TCP,9500/TCP 23m
```
##### PVC service cluster IP address
```powershell
$pv_ip=kubectl -n openebs get svc $volume_name -o jsonpath='{.spec.clusterIP}'
$pv_ip
```
Results:
```text
172.21.152.42
```

#### Get Pool Pod name

```powershell
kubectl -n openebs get po
```
Results:
```text
NAME                                                            READY STATUS  RESTARTS AGE
cstor-disk-pool-fw3x-8446689d5-sjbh6                            3/3   Running 0        9h
maya-apiserver-68f79c6c8-r45lb                                  1/1   Running 3        10h
openebs-admission-server-6fdbbff64c-hxh4f                       1/1   Running 0        10h
openebs-localpv-provisioner-6648755679-ptv8x                    1/1   Running 0        10h
openebs-ndm-operator-585d97db4b-qp8sg                           1/1   Running 1        10h
openebs-ndm-z586v                                               1/1   Running 0        8h
openebs-provisioner-5d8fccf8cc-w94p9                            1/1   Running 0        10h
openebs-snapshot-operator-658ccc99b7-v99th                      2/2   Running 0        10h
pvc-e9a4ed7c-aa68-440a-88cf-fc799b3781a9-target-7657858bdfmltcm 3/3   Running 1        25m
```

The Pool Pod name is "cstor-disk-pool-fw3x-8446689d5-sjbh6"

#### Modify iSCSI target IP in Pool Pod

```poweshell
kubectl -n openebs exec -it (kubectl -n openebs get po -l app=cstor-pool -o jsonpath='{.items[0].metadata.name}') -c cstor-pool -- bash
```
```bash
zfs get io.openebs:targetip
```
Results:
```text
NAME                                                                                                PROPERTY            VALUE SOURCE
cstor-e1ddb29d-0561-4dd4-be9e-6e46df3347c5                                                          io.openebs:targetip -     -
cstor-e1ddb29d-0561-4dd4-be9e-6e46df3347c5/pvc-e9a4ed7c-aa68-440a-88cf-fc799b3781a9                 io.openebs:targetip       default
cstor-e1ddb29d-0561-4dd4-be9e-6e46df3347c5/pvc-e9a4ed7c-aa68-440a-88cf-fc799b3781a9@nginx-backup-pv io.openebs:targetip -     -
```
##### Modify targetip to $pv_ip pool/$volume_name

Please type "zfs set io.openebs:targetip=$pv_ip pool/$volume_name" in the shell.

```bash
zfs set io.openebs:targetip=172.21.152.42 cstor-e1ddb29d-0561-4dd4-be9e-6e46df3347c5/pvc-e9a4ed7c-aa68-440a-88cf-fc799b3781a9
zfs get io.openebs:targetip
```
Results:
```text
NAME                                                                                                PROPERTY             VALUE         SOURCE
cstor-e1ddb29d-0561-4dd4-be9e-6e46df3347c5                                                          io.openebs:targetip  -             -
cstor-e1ddb29d-0561-4dd4-be9e-6e46df3347c5/pvc-e9a4ed7c-aa68-440a-88cf-fc799b3781a9                 io.openebs:targetip  172.21.152.42 local
cstor-e1ddb29d-0561-4dd4-be9e-6e46df3347c5/pvc-e9a4ed7c-aa68-440a-88cf-fc799b3781a9@nginx-backup-pv io.openebs:targetip  -             -
```
Exit shell
```bash
exit
```

## Check restore

```powershell
kubectl -n nginx-example get po -l app=nginx       
```
Results:
```text
NAME                              READY STATUS  RESTARTS AGE
nginx-deployment-5ccc99bffb-kdqc2 2/2   Running 0        48m
```

## Retrieve access.log

```powershell
kubectl -n nginx-example exec -it (kubectl -n nginx-example get pod -l "app=nginx" -o jsonpath='{.items[0].metadata.name}') -c nginx -- cat /var/log/nginx/access.log
```
Results:
```text
127.0.0.1 - - [28/Jul/2020:04:19:18 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.55.1" "-"
127.0.0.1 - - [28/Jul/2020:04:21:52 +0000] "GET /error HTTP/1.1" 404 153 "-" "curl/7.55.1" "-"
```

## Delete example application

```powershell
kubectl delete ns nginx-example
```
Results:
```text
namespace "nginx-example" deleted
```
