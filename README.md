## Quickstart Hashicorp Vault in IBM Cloud
This document provides instructions to install Hashicorp Vault in IBM Cloud Kubernetes service (IKS). Information here is to get started with hosting Vault service on IBM Cloud, but should not be used to design and provision a production Vault deployment. Information in this document is based on [hashicorp vault guide](https://github.com/hashicorp/vault-guides/tree/master/operations/provision-vault/kubernetes/minikube).


##### Create IKS cluster
1. [Log in](https://cloud.ibm.com/login) to your IBM Cloud account. If you do not have an account, you can [register](https://cloud.ibm.com/registration).  

2. [Create](https://cloud.ibm.com/kubernetes/catalog/cluster/create) an IKS free or standard cluster.

##### Setup IBM Cloud CLI
- Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl) and [ibmcloud](https://cloud.ibm.com/docs/cli/reference/ibmcloud?topic=cloud-cli-install-ibmcloud-cli#shell_install) CLIs.  

- Install container-service plugin  
```
$ ibmcloud plugin install container-service -r 'IBM Cloud'
```  

- Log into IBM Cloud CLI  
```
$ ibmcloud login -a https://cloud.ibm.com -r <region> -g <resource-group>  

    where -r is region, such as 'us-south' or 'eu-gb'  
          -g is resource-group name where your IKS cluster was created  
```  

- Initialize IKS plugin  
```
$ ibmcloud ks init
```  

- Set kubectl context to IKS cluster config  
```
$ eval $(ibmcloud ks cluster-config <your-iks-cluster-name> |tail -2|head -1)
```  

- Verify kubectl is set to manage your IKS cluster  
```
$ kubectl config current-context
```

> NOTE: Do not close this terminal session. Use this session to run all the commands in this document.  

##### Install Hashicorp Vault OSS
- Create a kubernetes namespace named `vault`. All vault resources will be created in this namespace.  
```
$ kubectl create namespace vault
```

- Set namespace in current context to `vault`.
```
$ kubectl config set-context $(kubectl config current-context) --namespace vault
```

- Create Storage backend for Vault.
```
$ kubectl apply -f https://raw.githubusercontent.com/ssibm/iks-hashicorp-vault/master/deploy/persistence.yaml
```

- Confirm the persistent volume claim is `Bound` to the persistent volume.
```
$ kubectl get pvc/vault-pvc -o jsonpath='{.status.phase}'
```

- Deploy Vault as a StatefulSet.
```
$ kubectl apply -f https://raw.githubusercontent.com/ssibm/iks-hashicorp-vault/master/deploy/config.yaml
$ kubectl apply -f https://raw.githubusercontent.com/ssibm/iks-hashicorp-vault/master/deploy/statefulset.yaml
```

- Verify Hashicorp Vault is running.
```
$ kubectl get statefulset/vault-ss

  NAME       DESIRED   CURRENT   AGE
  vault-ss   1         1         32s
```

- Create the services to expose Vault UI and API endpoints.
```
$ kubectl apply -f https://raw.githubusercontent.com/ssibm/iks-hashicorp-vault/master/deploy/services.yaml
```

- Check status of the running Vault service. Inspect the output, it should show your Vault instance is _Not Initialized_ and is _Sealed_.
```
$ kubectl exec -it vault-ss-0 -c vault -- /bin/sh -c 'vault status'

    Key                Value
    ---                -----
    Seal Type          shamir
    Initialized        false
    Sealed             true
    Total Shares       0
    Threshold          0
    Unseal Progress    0/0
    Unseal Nonce       n/a
    Version            n/a
    HA Enabled         false
```

##### Initialize Vault
> __Save the Initial Root Token and all the Unseal Keys in a safe place. These will be required everytime you have to run the unseal operation.__  

Run the following command to initialize the Vault service with default number of unseal key shares and number of keys required to unseal.
```
$ kubectl exec -it vault-ss-0 -c vault -- /bin/sh -c 'vault operator init'
```

Your output should look like below, displaying the Unseal keys and Initial Root Token. Save the token and unseal keys. These will be required to run Unseal operation and access your Vault.  
```
Unseal Key 1: ALQcPBI/ebuRe5n2mbef0G0coezwJw22aJYJ8Bzon/dE
Unseal Key 2: eC8wcjdvuScxwz2VJ3te85qTBUzu74qXrE2N0vAhqKfr
Unseal Key 3: pWL/dczqNHqZhTf02LkLEvhrj14/65sX94sZwKO1psyF
Unseal Key 4: HQuX3UHiJnUI7Q2Ja60E1pKVlrHbyYdi9DoRNquIek7X
Unseal Key 5: wEZY2rpnqyigqwfolG9RN/BtHKMKzZbir/yFJPgcdCW5  

Initial Root Token: s.tRIHHYZlndKwtsnqgupEE77E  

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!
```

Check your Vault status again to confirm it is __Initialized__.
```
$ kubectl exec -it vault-ss-0 -c vault -- /bin/sh -c 'vault status'

  Key                Value
  ---                -----
  Seal Type          shamir
  Initialized        true
  Sealed             true
  Total Shares       5
  Threshold          3
  Unseal Progress    0/3
  Unseal Nonce       n/a
  Version            1.1.0
  HA Enabled         false
```

##### Unseal Vault
By default, Vault initializes with 5 unseal keys and requires any 3 of those 5 keys to unseal.

__Total Shares__ from vault status shows the number of unseal keys, and __Threshold__ value is the number of unseal keys required to successfully complete the vault unseal operation.

Run the unseal command meeting the threshold volue, each time with a different unseal key. Track the __Unseal Progress__ from the command output to ensure unseal is progressing.

Unseal with first key. Unseal Progress will change to 1/3 (if Threshold is equal to 3)
```
$ kubectl exec -it vault-ss-0 -c vault -- /bin/sh -c 'vault operator unseal ALQcPBI/ebuRe5n2mbef0G0coezwJw22aJYJ8Bzon/dE'

  Key                Value
  ---                -----
  Seal Type          shamir
  Initialized        true
  Sealed             true
  Total Shares       5
  Threshold          3
  Unseal Progress    1/3
  Unseal Nonce       929b70a1-f292-fff7-4990-6b6d40729482
  Version            1.1.0
  HA Enabled         false
```

Run unseal command again with second key. Unseal progress will change to 2/3.
```
$ kubectl exec -it vault-ss-0 -c vault -- /bin/sh -c 'vault operator unseal eC8wcjdvuScxwz2VJ3te85qTBUzu74qXrE2N0vAhqKfr'

  Key                Value
  ---                -----
  Seal Type          shamir
  Initialized        true
  Sealed             true
  Total Shares       5
  Threshold          3
  Unseal Progress    2/3
  Unseal Nonce       929b70a1-f292-fff7-4990-6b6d40729482
  Version            1.1.0
  HA Enabled         false
```

Run the unseal command again. Repeat this until you meet the threshold. After the last unseal key is used, the output will no longer display _Unseal Progress_, but will instead show the value of __Sealed__ is __false__ indicating the Vault is now unsealed and ready for use.
```
$ kubectl exec -it vault-ss-0 -c vault -- /bin/sh -c 'vault operator unseal pWL/dczqNHqZhTf02LkLEvhrj14/65sX94sZwKO1psyF'

  Key             Value
  ---             -----
  Seal Type       shamir
  Initialized     true
  Sealed          false
  Total Shares    5
  Threshold       3
  Version         1.1.0
  Cluster Name    vault-cluster-4de9f22a
  Cluster ID      4d594db9-4ab5-1849-503c-501649d51f2c
  HA Enabled      false
```

Anytime the Vault is in Sealed state, repeat above steps to Unseal. Vault service can also be unsealed using the Vault UI web application.

##### Access Vault UI
- Get the Vault UI service URL. Vault HTTP API is also available at the same URL for use by other services over the public network. For all services running on the same IKS cluster, Vault HTTP API service is available at __vault:8200__.
```
$ kubectl get svc/vault-ui -o jsonpath='http://{.status.loadBalancer.ingress[].ip}:{.spec.ports[].port}'
```

- Open Vault UI URL in web browser. If the Vault service is in Sealed state, enter one Unseal key at a time to unseal the vault.

- Set login method to _Token_ and enter the initial root token to log into the vault service.

- You can now begin using Vault for secrets management. Refer to [Hashicorp Vault documentation](https://www.vaultproject.io/docs/secrets/index.html) on how to use Vault.

##### Uninstall Vault resources
Run following commands to delete all the Vault resources created using this document.
```
$ kubectl delete namespace vault
$ kubectl delete pv/vault-pv
$ kubectl config set-context $(kubectl config current-context) --namespace default
```

Hashicorp Vault is now running in IBM Cloud. Refer to [Vault Reference Architecture](https://learn.hashicorp.com/vault/day-one/ops-reference-architecture) to design a highly-available Vault Enterprise service for production deployment in IBM Cloud.

##### References
- [Introduction to Vault](https://www.vaultproject.io/docs/what-is-vault/index.html)
- [Hashicorp Vault Concepts - Seal/Unseal](https://www.vaultproject.io/docs/concepts/seal.html)
- [IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers?topic=containers-ibm-cloud-kubernetes-service-technology#ibm-cloud-kubernetes-service-technology)
- [IBM Cloud Object Storage](https://cloud.ibm.com/docs/services/cloud-object-storage?topic=cloud-object-storage-administrators#administrators)
- [IBM Cloud File Storage](https://cloud.ibm.com/docs/infrastructure/FileStorage?topic=FileStorage-about#getting-started-with-file-storage)
