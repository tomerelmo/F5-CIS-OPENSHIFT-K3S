# Here we are installing the ipam on the openshift cluster
### ipam is conatiner that inchagre of IP ALLOCATION across the virual services we are deploying
### we are configuring ipam with predefined ranges that we later mention as labels to the virtual server we are creating 
### connect to web shell on ocp-provisioner server
![picture of the ocp-provisioner](/img/02-1.png)
### connect to the "web shell" and login as cloud-user
```text
su cloud-user

# make yourself a folder which you going to work with 
mkdir /home/cloud-user/ipam
cd /home/cloud-user/ipam
```
### add the f5 ipam controller helm to the chart :
```text
helm repo add f5-ipam-stable https://f5networks.github.io/f5-ipam-controller/helm-charts/stable

# make sure its added to the local charts 
#"f5-ipam-stable" has been added to your repositories
```
### we will change parametes for the values yaml , this will create the neccesery components for bring up the IPAM :
```text
wget -O value.yaml https://raw.githubusercontent.com/F5Networks/f5-ipam-controller/main/helm-charts/f5-ipam-controller/values.yaml
```

#### for installing the ipam we will need several components which the chart defened with ( we just making the changes we need) , such as - RBAC , ServiceAccount, persistance volume and image version (also from where to pull)

#### edit the following lines on the values to prepare for applying

#### change the ip range ip part 
```text

#find the following line :  
# REQUIRED Params if provider is f5-ip-provider
  ip_range: '{"test":"172.16.1.1-172.16.1.5", "prod":"172.16.1.50-172.16.1.55"}'

#replace the range line with the following line:
  ip_range: '{"test":"10.1.10.100-10.1.10.120", "prod":"10.1.10.130-10.1.10.150"}'

```

#### we need also to change the persist volume configurations, we will have to create persistence 

```text
#set pvc creation for ture
pvc:
  # set create tag to true to create new persistent volume claim and set storageClassName,accessMode and storage
  create: true

  #name of the  persistent volume claim to be used
  # If not set and create is true, a name is generated using the fullname template
  name:

  #if create set to false below parameters will be ignored
  storageClassName: openebs-hostpath
  accessMode: ReadWriteOnce
  storage: 1Gi

```
####  we will have to push the image of the ipam to the registry to be avialable on the openshift
```text
#change the following for use the local internal images 
image: 
  # Use the tag to target a specific version of the Controller
  user: docker.io/f5networks  #add docker.io as we will download from docker.io the image and we will pull localy
  repo: f5-ipam-controller
  pullPolicy: IfNotPresent #this from always to ifNotPresent
  version: 0.1.12 #this from 0.1.5 to 0.1.12


  user: docker.io/f5networks
  repo: f5-ipam-controller
  pullPolicy: IfNotPresent
  version: 0.1.12
```


#### lets login into the each node , pull the image and create the folder for the PVC
```text
oc debug node/worker-1.ocp.f5-udf.com - to login into the node
# wait few seconds
# while you see the following :
#sh-5.1#
# type:
chroot /host

# now download the ipam image to the podman :
podman pull docker.io/f5networks/f5-ipam-controller:0.1.12

# do the same for worker2 
oc debug node/worker-2.ocp.f5-udf.comß

```

#### lets create local persistence volume on worker1
```text
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ipam-local-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: ipam
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /var/f5-ipam
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
            - worker-1.ocp.f5-udf.com

# create the folder on the worker node :
[cloud-user@ocp-provisioner ipam]$ oc debug node/worker-1.ocp.f5-udf.com
Starting pod/worker-1ocpf5-udfcom-debug-g4q2x ...
To use host binaries, run `chroot /host`. Instead, if you need to access host namespaces, run `nsenter -a -t 1`.
Pod IP: 10.1.10.9
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1# mkdir -p /var/f5-ipam
chmod 777 /var/f5-ipam
exit
exit

[cloud-user@ocp-provisioner ipam]$ oc apply -f pvc.yaml 
persistentvolume/ipam-local-pv created


```

#### In production, the F5 IPAM Controller must use a dynamically provisioned PersistentVolume backed by a resilient CSI storage class (e.g., Ceph RBD, cloud block storage, or enterprise SAN) with volumeBindingMode: WaitForFirstConsumer, ensuring the IPAM database is durable, node-independent, and survives pod restarts or node failures.

#### run 
```text
oc get pods -A | grep ipam

# and make sure the pod is running and ready

kube-system                                        ipam-f5-ipam-controller-5f766c75cf-ncgpc                          1/1     Running     0          5m30s
```
#### we will configure the ipam values such as the addresses ranges and the label for each range (there are DEV and PROD)
```text
'{"test":"172.16.1.1-172.16.1.5", "prod":"172.16.1.50-172.16.1.55"}'
```

#### we configured the ranges so, when we will deploy new VS CRD object, we will mention the label (which is either prod or test) and the VirtualServer on teh BIGIP will get ip by the ipam allocation related to the pool test or prod

