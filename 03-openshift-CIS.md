
### Now that we have the ipam installed, we can move on to install the CIS 

# download the repo of the cis
```text
git clone https://github.com/F5Networks/k8s-bigip-ctlr.git

cd k8s-bigip-ctlr/helm-charts/f5-bigip-ctlr/

```

#### before we starting , lets pull the images into the podman of the workers 
```text
# for worker 1
oc debug node/worker-1.ocp.f5-udf.com -- chroot /host podman pull docker.io/f5networks/k8s-bigip-ctlr:2.20.3

# for worker 2
oc debug node/worker-2.ocp.f5-udf.com -- chroot /host podman pull docker.io/f5networks/k8s-bigip-ctlr:2.20.3
```


#### now we will apply the helm chart with changes (--set ) its simply means to override the defalut values under values.yaml
#### we are setting the bigip url and the GTM (dns) url to be Float Sefl ip of the bigip ( so if failover occurs, the system will be resilient)
```text
helm install cis ./ --set bigip_secret.create="true" --set bigip_secret.username="admin" --set bigip_secret.password="F5train1!" --set args.bigip_url="https://10.1.30.201" --set  args.bigip_partition="k8s" --set args.custom-resource-mode="true"  --set args.insecure="true" --set args.gtm-bigip-password="F5train1!" --set args.gtm-bigip-username="admin" --set args.gtm-bigip-url="https://10.1.30.201" --set args.pool_member_type="cluster" --set image.user="docker.io/f5networks" --set image.repo="k8s-bigip-ctlr" --set image.pullPolicy="IfNotPresent"  --set version="2.20.3" --set args.ipam="true"

 # make sure the chart created

 NAME: cis
LAST DEPLOYED: Tue Mar  3 09:13:36 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Container Ingress Services controller: cis

```

#### we can see the the conatiner is not ready yet and its on loop crash, lets find out with logs whats is going on:
```text
oc get pods -A | grep cis

#take your cis pods name and logs it
[cloud-user@ocp-provisioner /]$ oc logs -n kube-system cis-f5-bigip-ctlr-56584bdf9f-hjzbk
2026/03/03 18:04:57 [WARNING] BIG-IP credentials are not set. Checking back to Creds Directory.
2026/03/03 18:04:57 [INFO] [INIT] Starting: Container Ingress Services - Version: v2.20.3, BuildInfo: gitlab-16324349-acadb9e63a1117defbb70d6096950df1fe5db713
2026/03/03 18:04:58 [ERROR] [AS3][BigIP] RPM is not installed on BIGIP, Error response from BIGIP with status code 404 
2026/03/03 18:04:58 [ERROR] AS3 RPM is not installed on BIGIP
```
#### the above error is mentioning that we have to install AS3 automation system into the bigip . visit the following url and download the rpm
```text
https://github.com/F5Networks/f5-appsvcs-extension/releases/download/v3.56.0/f5-appsvcs-3.56.0-10.noarch.rpm
```

#### go into BIGIP-OC-01 on the components section and go into TMUI on the login scren enter
```text
user: admin
password: F5train1!
```
![bigip login page](/img/03-bigip-login.png)

#### navigate into iapp and Package Management LX 
![iapp mgmt package lx](/img/03-package-management-lx.png)

#### import the rpm package - press import and navigate to the folder which you downloaded the package
![import AS23](/img/03-import-as3.png)

#### do the upload for all the bigip on this lab:
```text
BIGIP-OC-01
BIGIP-OC-02
BIGIP-K3S-01
BIGIP-K3S-02
```

#### now lets check again the status of the container 
```text
[cloud-user@ocp-provisioner /]$ oc get pods -A | grep -i cis
kube-system                                        cis-f5-bigip-ctlr-56584bdf9f-hjzbk                                0/1     Running     10 (6m9s ago)   27m
```

#### note: its take some time to the container to be ready, you may wait serverl minutes 
```text
[cloud-user@ocp-provisioner /]$ oc get pods -A | grep -i cis
kube-system                                        cis-f5-bigip-ctlr-56584bdf9f-hjzbk                                1/1     Running     10 (7m25s ago)   28m
```

#### now , we may test the solution, lets make some demo deployment and service with type - cluster ip
```text
# lets pull the image on worker 1
oc debug node/worker-1.ocp.f5-udf.com -- chroot /host podman pull docker.io/nginx:1.25

# pull the image on worker 2
oc debug node/worker-2.ocp.f5-udf.com -- chroot /host podman pull docker.io/nginx:1.25

# copy the following yaml that will be instantly apply 

cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: docker.io/nginx:1.25
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: nginx-demo
  ports:
  - port: 80
    targetPort: 80
EOF



# make sure that the deployment and the service created :
deployment.apps/nginx-demo created
service/nginx-demo created

# make sure you can see the pod on ready state
oc get pods -A | grep nginx
```

#### navigate into the following github to see custom resource example of the large amout of possibilites that we can configure for creating VS 
```text
https://github.com/F5Networks/k8s-bigip-ctlr/tree/master/docs/config_examples/customResource
```

#### pay attention to the different components that there is such as - virtualServer (for http), transportServer ( for none http), VirtualServerWithTLSProfile , ExternalDNS ( we will visit it on gslb)

#### now lets make CRD object Virtual server and apply it
#### Note: we are applying here the IPAM , we are taking one of the test range ip address
```text
cat <<'EOF' | oc apply -f -
apiVersion: cis.f5.com/v1
kind: VirtualServer
metadata:
  name: nginx-demo-vs
  namespace: default
  labels:
    f5cr: "true"
spec:
  host: nginx-demo.cis.lab
  ipamLabel: test
  httpTraffic: "allow"
  snat: auto
  pools:
    - path: /
      service: nginx-demo
      servicePort: 80
      monitors:
        - type: http
          interval: 10
          timeout: 10
          send: "GET / HTTP/1.1\r\nConnection: close\r\n\r\n"
          recv: ""
          targetPort: 80
EOF
```

#### looks like we have some error :
```text
error: resource mapping not found for name: "nginx-demo-vs" namespace: "default" from "STDIN": no matches for kind "VirtualServer" in version "cis.f5.com/v1"
ensure CRDs are installed first
```

#### we have to install the customerResourceDefinition to make oc know what we are talking about while we are deploying VirtualServer kind 
```text
[cloud-user@ocp-provisioner /]$ cd /home/cloud-user/cis/k8s-bigip-ctlr/docs/config_examples/customResourceDefinitions/

[cloud-user@ocp-provisioner customResourceDefinitions]$ oc apply -f customresourcedefinitions.yml 
customresourcedefinition.apiextensions.k8s.io/virtualservers.cis.f5.com created
customresourcedefinition.apiextensions.k8s.io/tlsprofiles.cis.f5.com created
customresourcedefinition.apiextensions.k8s.io/transportservers.cis.f5.com created
customresourcedefinition.apiextensions.k8s.io/externaldnses.cis.f5.com created
customresourcedefinition.apiextensions.k8s.io/ingresslinks.cis.f5.com created
customresourcedefinition.apiextensions.k8s.io/policies.cis.f5.com created
```

#### now lets try again :
```text
cat <<'EOF' | oc apply -f -
apiVersion: cis.f5.com/v1
kind: VirtualServer
metadata:
  name: nginx-demo-vs
  namespace: default
  labels:
    f5cr: "true"
spec:
  host: nginx-demo.cis.lab
  ipamLabel: test
  snat: auto
  pools:
    - path: /
      service: nginx-demo
      servicePort: 80
      monitors:
        - type: http
          interval: 10
          timeout: 10
          send: "GET / HTTP/1.1\r\nConnection: close\r\n\r\n"
          recv: ""
          targetPort: 80
EOF
```

#### make sure the VS created and working
```text
[cloud-user@ocp-provisioner customResourceDefinitions]$ oc get vs
Warning: short name "vs" could also match lower priority resource volumesnapshots.snapshot.storage.k8s.io
NAME            HOST                 TLSPROFILENAME   HTTPTRAFFIC   IPADDRESS   IPAMLABEL   IPAMVSADDRESS   STATUS   AGE
nginx-demo-vs   nginx-demo.cis.lab                                              test        10.1.10.100     OK       44s
[cloud-user@ocp-provisioner customResourceDefinitions]$ 
```


#### now let have a look on the Virtual Server created on the bigip- navigate to "Local Traffic" and then to Virtual servers
![virtual server under local traffic](/img/03-ltm-vs.png)


### Were is the Virtual Server (VS) we created ?
#### we are configuring the CIS to work with some auth partition so we will have to change the partition on the right up cornner to the partition we configured on the Helm values 

```text
--set  args.bigip_partition="k8s"
```

#### lets navigate into the partition and then we can see our Virtual server configured
![partition](/img/03-partition.png)

#### we are currently on bigip 02 ( this is the active one currenly)
![bigip 02](/img/03-bigip02.png)

#### lets go to the components lab section and press on TMUI , bring up the manamgemtn user interface of BIGIP-OC-01 (if bigip01 is the one that active for you , then you need to login to 02)

#### switch to the right partiton as we learned,can you see the Virtual server ?

#### the configuration now is synced manually , to sync the configuration automatically, login to the active bigip TMUI and go the Devices Management --> Device Group And press on OC_Cluster
![device-mgmt](/img/03-device-mgmt.png)

#### Swithc from manualy sync into automatic sync
![device-mgmt](/img/03-autosync.png)

#### now lets delete the VS we created on openshift , make sure that we cannot see the VS on both BIGIP and then run the VS again to make sure we see the VS on both
```text
[cloud-user@ocp-provisioner k8s-bigip-ctlr]$ oc delete vs nginx-demo-vs
Warning: short name "vs" could also match lower priority resource volumesnapshots.snapshot.storage.k8s.io
virtualserver.cis.f5.com "nginx-demo-vs" deleted
```

#### now make sure you dont see any of the VS configured on both devices , now run again the VS creation command , and make sure you are seeing the VS on both devices
```text
cat <<'EOF' | oc apply -f -
apiVersion: cis.f5.com/v1
kind: VirtualServer
metadata:
  name: nginx-demo-vs
  namespace: default
  labels:
    f5cr: "true"
spec:
  host: nginx-demo.cis.lab
  ipamLabel: test
  snat: auto
  pools:
    - path: /
      service: nginx-demo
      servicePort: 80
      monitors:
        - type: http
          interval: 10
          timeout: 10
          send: "GET / HTTP/1.1\r\nConnection: close\r\n\r\n"
          recv: ""
          targetPort: 80
EOF
```
#### now the configuration synced automatically so each change we made will affect the other bigip immediatly - there are down sides to this design and up sides , can you tell which benefits and which down sides we have ?

#### now lets test some traffic to the nginx-demo.cis.lab application which is on ip address 10.1.10.100 - we cant , because the bigip cant forward the traffic to the pod ip
```text
root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# list ltm node /k8s/10.244.3.70 
ltm node /k8s/10.244.3.70 {
    address 10.244.3.70
    partition k8s
}
root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# ping 10.244.3.70
PING 10.244.3.70 (10.244.3.70) 56(84) bytes of data.
^C
--- 10.244.3.70 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms
```

#### but the bigip done know the networks 10.244.3.0/24 or 10.244.4.0/24 which for pod subnet for worker1 and worker2, lets just define default route towards them via the cis_data interface , so we will have to oute subnet 10.244.4.0/24 to 10.1.10.10 and subnet 10.244.3.0/24(worker 1) to 10.1.10.9

#### on 1 of the bigip cluster , navigate to Network then to Route,
![](/img/03-route-network.png)

#### press add on righ corrner and add the following :
![](/img/03-add-worker-pod-gw.png)

#### dont forget to do the same for 10.244.4.0/24 via 10.1.10.10

#### make sure you can reach to both workers pods ip from the WebShell
```text
root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# ping 10.244.3.70
PING 10.244.3.70 (10.244.3.70) 56(84) bytes of data.
64 bytes from 10.244.3.70: icmp_seq=1 ttl=62 time=2.11 ms
^C
--- 10.244.3.70 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 2.116/2.116/2.116/0.000 ms
root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# ping 10.244.4.19
PING 10.244.4.19 (10.244.4.19) 56(84) bytes of data.
64 bytes from 10.244.4.19: icmp_seq=1 ttl=62 time=3.05 ms
```


#### now lets go again to the ocp provioner and test curl traffic to the VS we created
```text
[cloud-user@ocp-provisioner k8s-bigip-ctlr]$ curl http://10.1.10.100 -H "Host: nginx-demo.cis.lab"
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

#### so here we are having a pair of bigip on HA that have the same virtual server configurations , they are fault tolerrent and backup each other  


#### login to webshell of both of the BIGIP-OC pair
![webshell](/img/03-webshell.png)
#### enter the following command to search where is the active BIGIP:
```text
[root@BIGIP-OC-02:Active:In Sync] config # tmsh   
root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# show sys failover
Failover active for 1d 23:19:21
root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# 
```

#### restart the active bigip and immediatly test the if the application still working from the web shell of "OCP-Provisioner"
```
# on active BIGIP-OC :
root@(BIGIP-OC-02)(cfg-sync In Sync)(Active)(/Common)(tmos)# quit
[root@BIGIP-OC-02:Active:In Sync] config # reboot
Terminated

# immedialty check the application from the ocp-provisioner
curl http://10.1.10.100 -H "Host: nginx-demo.cis.lab"
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to

# check that the other bigip is active
root@(BIGIP-OC-01)(cfg-sync Disconnected)(Active)(/Common)(tmos)# show sys failover
Failover active for 0d 00:01:12
root@(BIGIP-OC-01)(cfg-sync Disconnected)(Active)(/Common)(tmos)# 

# see that the othe bigip active for 1 minute and 12 seconds only
```
