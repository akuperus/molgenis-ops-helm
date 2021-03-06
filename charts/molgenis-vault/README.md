# MOLGENIS Vault helm chart

This chart creates a vault operator, but NO vault.

## Deploy
The deployment of the Vault consists of 7 steps.

**In Rancher**
1. Delete molgenis-vault app in prod
2. Deploy molgenis-vault in prod to the vault-operator namespace

**On your own machine**

*Context == prod-molgenis*

3. *Create Vault service*

   ```rancher kubectl create -f resources/vault.yaml --namespace vault-operator```
   
   See https://github.com/coreos/vault-operator/blob/master/doc/user/vault.md
   
   >**IMPORTANT**: ```core: security barrier not initialized``` in logging is normal. Please restore the backup first and unseal the vault.

4. *Add NodePort service to expose to other clusters*

   Create a NodePort service on top of the Vault service
   ```rancher kubectl create -f resources/vault-nodeport.yaml --namespace vault-operator```

5. *Restore backup*

   Fill in the filename of the Minio backup ```<specify the backup name>``` in ```restore.yaml```
   ```rancher kubectl create -f resources/restore.yaml --namespace vault-operator```

6. *Unlock the Vault*

   There are 2 Vault services which need to be unlocked.
   ```rancher kubectl describe vault --namespace vault-operator```
   You need to look here:
   ```
   Vault Status:
     Active:
     Sealed:
       **vault-79f8d698fb-5thm2**
       **vault-79f8d698fb-wh8js**
     Standby:  <nil>
   ```
   Proxy-forward the Vault service to be able to make use of the client.
   ```rancher kubectl port-forward #vault-pod# 8200 --namespace vault-operator```
   Install Vault client: https://www.vaultproject.io/docs/install/
   You need to add this line to your ```.profile```: ```export VAULT_SKIP_VERIFY=1```
   Then execute the following commands:
   ```vault status```
   Exceute this three times (because of the three key encryption method)
   ```vault operator unseal```
   Run ```vault status``` again to check if the Vault is unsealed.

**In Jenkins**

7. The NodePort is random so you need to configure it in the Kubernetes secret 

   (Rancher --> dev-molgenis project): ```molgenis-pipeline-vault-secret``` with key ```addr```. E.g. 'https://192.168.64.161:30221'.
   (Go to the ```prod-molgenis``` project and select **Resources** --> **Secrets**).

## Parameters

### Minio credentials
Define credentials for backup to the Azure Blob Store.
See [etcd-operator documentation](https://github.com/coreos/etcd-operator/blob/master/doc/user/abs_backup.md).

| Parameter             | Description                   | Default            |
| --------------------- | ----------------------------- | ------------------ |
| `aws.accessKeyId`     | name accessKeyId Minio        | `xxxx`             |
| `aws.secretAccessKey` | access secretKey Minio        | `xxxx`             |
| `aws.endpoint`        | endpoint Minio                | `Minio`            |

### Backup job
Define the schedule of the backup job

| Parameter            | Description                  | Default       |
| -------------------- | ---------------------------- | ------------- |
| `backupJob.enable`   | Enable backup cronjob        | `true`        |
| `backupJob.schedule` | cron schedule for the backup | `0 12 * * 1`  |

### UI

Parameter | Description | Default
--------- | ----------- | ------- 
`ui.replicaCount` | desired number of Vault UI pod | `1`
`ui.image.repository` | Vault UI container image repository | `djenriquez/vault-ui`
`ui.image.tag` | Vault UI container image tag | `latest`
`ui.resources` | Vault UI pod resource requests & limits | `{}`
`ui.nodeSelector` | node labels for Vault UI pod assignment | `{}`
`ui.ingress.enabled` | If true, Vault UI Ingress will be created | `true`
`ui.ingress.annotations` | Vault UI Ingress annotations | `{}`
`ui.ingress.host` | Vault UI Ingress hostname | `vault.molgenis.org`
`ui.ingress.tls` | Vault UI Ingress TLS configuration (YAML) | `[]`
`ui.vault.url` | Vault UI default vault url | `https://vault.vault-operator:8200`
`ui.vault.auth` | Vault UI login method | `GITHUB`
`ui.service.name` | Vault UI service name | `vault-ui`
`ui.service.type` | type of ui service to create | `ClusterIP`
`ui.service.externalPort` | Vault UI service target port | `8000`
`ui.service.internalPort` | Vault UI container port | `8000`
`ui.service.nodePort` | Port to be used as the service NodePort (ignored if `server.service.type` is not `NodePort`) | `0`


