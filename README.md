# How to deploy Quay with RHOCS backend on OCP 4.6



## Introduction
Quay is a secure, private container registry that builds, analyzes and distributes container images. In this post we will show how we deploy it on top of OCP 4 and how we use OCS Multi-Cloud Object Gateway object service to provide a storage backend to Quay. We will also enable Clair container vulnerability analysis service.




## Lab Virtual Env Overview
* OCP 4.6 UPI baremetal deployment
* All oc commands are issued from a CentOS 8 bastion node
* 3 Masters (16Go of RAM) and 3 Workers (32Go of RAM) VMs
```bash
[root@bastion ~] oc get nodes
NAME                        STATUS   ROLES    AGE   VERSION
master0.mycluster.lab.local   Ready    master   10h   v1.19.0+7070803
master1.mycluster.lab.local   Ready    master   10h   v1.19.0+7070803
master2.mycluster.lab.local   Ready    master   10h   v1.19.0+7070803
worker0.mycluster.lab.local   Ready    worker   10h   v1.19.0+7070803
worker1.mycluster.lab.local   Ready    worker   10h   v1.19.0+7070803
worker2.mycluster.lab.local   Ready    worker   10h   v1.19.0+7070803
```
* OCS 4.6 has been deployed
 ```bash
 [root@bastion ~] oc get cephcluster -n openshift-storage
NAME                             DATADIRHOSTPATH   MONCOUNT   AGE     PHASE   MESSAGE                        HEALTH
ocs-storagecluster-cephcluster   /var/lib/rook     3          5h17m   Ready   Cluster created successfully   HEALTH_OK

 ```

## Simple deployment
* Let's  Create a dedicated Namespace and deploy Quay Operator
  ```bash
  cat << EOF | oc apply -f -
  apiVersion: v1
  kind: Namespace
  metadata:
    labels:
      openshift.io/cluster-monitoring: "true"
    name: local-quay
  ---
  apiVersion: operators.coreos.com/v1
  kind: OperatorGroup
  metadata:
    name: local-quay
    namespace: local-quay
  spec:
    targetNamespaces:
    - local-quay
  ---
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    name: quay-operator
    namespace: local-quay
  spec:
    channel: quay-v3.3
    installPlanApproval: Automatic
    name: quay-operator
    source: redhat-operators
    sourceNamespace: openshift-marketplace
  EOF

* We wait for the operator to be ready
```bash
[root@bastion ~] oc get pod -n local-quay
NAME                                                    READY   STATUS    RESTARTS   AGE
quay-operator-7cb56c7c5f-7dgwk                          1/1     Running   0          6s
```

* Create an OpenShift secret to be able to pull the required container images
```bash
oc create -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: redhat-pull-secret
data:
  .dockerconfigjson:  <Quay Key>
type: kubernetes.io/dockerconfigjson
EOF
Get the key here https://access.redhat.com/solutions/3533201
```


* RHOCS Object storage service provides an S3 API and is based on NooBaa project.

  To interact with the service, we need to:

  * Install Noobaa CLI on our Bastion

   ```bash
   OS="linux"
   VERSION=$(curl -s https://api.github.com/repos/noobaa/noobaa-operator/releases/latest | jq -r '.name')
   curl -LO https://github.com/noobaa/noobaa-operator/releases/download/$VERSION/noobaa-$OS-$VERSION
   chmod +x noobaa-$OS-$VERSION
   mv noobaa-$OS-$VERSION /usr/local/bin/noobaa
    ```
    Full documentation can be found here: https://github.com/noobaa/noobaa-operator

  * Retrieve S3 API credentials and endpoint
    ```bash
    [root@bastion ~] noobaa status -n openshift-storage
    INFO[0000] CLI version: 2.3.0                           
    INFO[0000] noobaa-image: noobaa/noobaa-core:5.5.0       
    INFO[0000] operator-image: noobaa/noobaa-operator:2.3.0
    INFO[0000] Namespace: openshift-storage                 
     .
     .
     .
    #----------------#
    #- S3 Addresses -#
    #----------------#
    ExternalDNS : [https://s3-openshift-storage.mycluster.lab.local]
    ExternalIP  : []
    NodePorts   : [https://10.226.115.10:30795]
    InternalDNS : [https://s3.openshift-storage.svc:443]
    InternalIP  : [https://172.30.20.203:443]
    PodPorts    : [https://10.131.0.90:6443]
    #------------------#
    #- S3 Credentials -#
    #------------------#
    AWS_ACCESS_KEY_ID     : FSfXsNiWnva1lmDnzZ1c
    AWS_SECRET_ACCESS_KEY : r+RpYkH8Es81ZipMjngCmHUfMvIa4TLYOlTJwLgE
    #------------------#
    #- Backing Stores -#
    #------------------#
    NAME                           TYPE            TARGET-BUCKET                             PHASE   AGE         
    noobaa-default-backing-store   s3-compatible   nb.1609358827719.mycluster.lab.local   Ready   94h48m18s   
    #------------------#
    #- Bucket Classes -#
    #------------------#
    NAME                          PLACEMENT                                                             PHASE   AGE         
    noobaa-default-bucket-class   {Tiers:[{Placement: BackingStores:[noobaa-default-backing-store]}]}   Ready   94h48m18s   
    #-----------------#
    #- Bucket Claims -#
    #-----------------#
    No OBCs found.
    ```
 Take good notes of ExternalDNS, InternalDNS, AWS_ACCESS_KEY_ID and AWS_ACCESS_KEY_ID as we will need it to create our Quay Echosystem
 * Install s3cmd and create .s3cmd config  (This is not mandatory but makes manipulation of buckets and objects easier)
 ```bash
 [root@bastion ~] dnf install s3cmd -y
 [root@bastion ~] cat .s3cfg
[default]
access_key = FSfXsNiWnva1lmDnzZ1c
secret_key = r+RpYkH8Es81ZipMjngCmHUfMvIa4TLYOlTJwLgE
host_base = s3-openshift-storage.mycluster.lab.local  ##We use ExternalDNS retrieved with nooba command
host_bucket = s3-openshift-storage.mycluster.lab.local
check_ssl_certificate = False
check_ssl_hostname = False
use_http_expect = False
use_https = True
signature_v2 = True
signurl_use_https = True
```
 * Create a bucket for Quay
 ```bash
 [root@bastion ~]# s3cmd mb  s3://quay1
 Bucket 's3://quay1/' created
```

* We can now create a QuayEcosystem with the following command:
```bash
oc create  -f - <<EOF
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: quayecosystem1
spec:
  clair:
    enabled: true
    imagePullSecretName: redhat-pull-secret
  quay:
    imagePullSecretName: redhat-pull-secret
    registryBackends:
      - name: rhocs
        rhocs:
          hostname: s3.openshift-storage.svc.cluster.local:443
          accessKey: FSfXsNiWnva1lmDnzZ1c
          secretKey: r+RpYkH8Es81ZipMjngCmHUfMvIa4TLYOlTJwLgE
          bucketName: quay1
EOF
```

 Let's take a closer look :

 * First we create a QuayEcossytem object and name it quayecosystem1

   ```yaml
   apiVersion: redhatcop.redhat.io/v1alpha1
   kind: QuayEcosystem
   metadata:
     name: quayecosystem1
   ```

 * We specify we want to enable Clair and we use redhat-pull-secret so we can pull the necessary image
 ```yaml
   spec:
     clair:
       enabled: true
       imagePullSecretName: redhat-pull-secret
 ```

 * This is the most important part (see comments)
 ```yaml
 quay:
   imagePullSecretName: redhat-pull-secret
   registryBackends:
     - name: rhocs
       rhocs:    # We specify Quay we are using rhcos as storage backend
         hostname: s3.openshift-storage.svc.cluster.local:443 #Make sure InternalDNS and Port are specified here
         accessKey: FSfXsNiWnva1lmDnzZ1c
         secretKey: r+RpYkH8Es81ZipMjngCmHUfMvIa4TLYOlTJwLgE
         bucketName: quay1 # This is the bucket we created previously
```         

   * hostname field needs to be specified with the port otherwise you'll get the infamous error:
   ```bash
    'Failed to Validate Component: registry-storage Validation Failed: Invalid storage configuration: rhocs: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed (_ssl.c:727)'    


* After a few minutes we check all pods are up and running
```bash
[root@bastion ~] oc get pod
NAME                                               READY   STATUS    RESTARTS   AGE
quay-operator-7cb56c7c5f-7dgwk                     1/1     Running   0          4d5h
quayecosystem1-clair-77f7844f4f-75lh8              1/1     Running   0          3h7m
quayecosystem1-clair-postgresql-5c69478878-bfmtn   1/1     Running   0          3h7m
quayecosystem1-quay-84cbc66cbd-mdw4j               1/1     Running   0          3h8m
quayecosystem1-quay-config-5bf565888c-dtvpv        1/1     Running   0          3h9m
quayecosystem1-quay-postgresql-c7645f945-nzhrr     1/1     Running   0          3h9m
quayecosystem1-redis-68945f56b7-xk67g              1/1     Running   0          3h10m
```
* We can now find out the route and access Quay GUI at https://quayecosystem1-quay-local-quay.mycluster.lab.local (login: quay pass: password)
```bash
[root@esxi-bastion ~] oc get route
NAME                         HOST/PORT                                                      PATH   SERVICES                     PORT   TERMINATION            WILDCARD
quayecosystem1-quay          quayecosystem1-quay-local-quay.mycluster.lab.local                 quayecosystem1-quay          8443   passthrough/Redirect   None
quayecosystem1-quay-config   quayecosystem1-quay-config-local-quay.mycluster.lab.local          quayecosystem1-quay-config   8443   passthrough/Redirect   None

## To be continued
