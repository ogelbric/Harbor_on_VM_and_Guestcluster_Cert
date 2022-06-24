# 1) Install Harbor on VM in docker
# 2) Generate cert for guest cluster local Harbor image pull
# 3) Build Guest cluster
# 4) Use image from local harbor and create POD


## Harbor download / install 

### Download: 
```
https://github.com/goharbor/harbor/releases
```

### Move file and unpack
```
scp harbor-offline-installer-v2.1.5.tgz root@10.197.79.2:/tmp/.
mv harbor-offline-installer-v2.1.5.tgz /usr/local
cd /usr/local
tar xzvf harbor-offline-installer-v2.1.5.tgz
```
### Generate keys
```
openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Dallas/L=Texas/O=example/OU=Personal/CN=orfdns.lab.local" \
 -key ca.key \
 -out ca.crt

openssl genrsa -out orfdns.lab.lab.key 4096

openssl req -sha512 -new \
    -subj "/C=CN/ST=Dallas/L=Texas/O=example/OU=Personal/CN=orfdns.lab.lab" \
    -key orfdns.lab.lab.key \
    -out orfdns.lab.lab.csr

cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=orfdns.lab.lab
DNS.2=lab.lab
DNS.3=orfdns
DNS.4=10.197.79.2
EOF

openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in orfdns.lab.lab.csr \
    -out orfdns.lab.lab.crt

mkdir -p /data/cert

cp orfdns.lab.lab.crt /data/cert/
cp orfdns.lab.lab.key /data/cert/

openssl x509 -inform PEM -in orfdns.lab.lab.crt -out orfdns.lab.lab.cert

mkdir -p /etc/docker/certs.d/orfdns.lab.lab

cp orfdns.lab.lab.cert /etc/docker/certs.d/orfdns.lab.lab/
cp orfdns.lab.lab.key /etc/docker/certs.d/orfdns.lab.lab/
cp ca.crt /etc/docker/certs.d/orfdns.lab.lab/

systemctl restart docker
```

### Harbor yaml file fix up
```
cp harbor.yml.tmpl  harbor.yml
Items changed in file
	hostname: orfdns.lab.lab
	certificate: /data/cert/orfdns.lab.local.crt
  	private_key: /data/cert/orfdns.lab.local.key
	harbor_admin_password: VMware1!
```

### Harbor install
```
sudo ./install.sh

Optional installation with Notary, Trivy, and Chart Repository Service
sudo ./install.sh --with-notary --with-trivy --with-chartmuseum
```

### Incase the harbor yaml file had some typos (in my case things need to be re-done)
```
Re-read / re-install harbor and harbor.yml file
--------------------------------------------------
cd /usr/local/harbor
sudo docker-compose down -v
sudo ./prepare
sudo docker-compose up -d

Clean install
---------------
cd /usr/local/harbor
sudo docker-compose down -v
rm -r /data/database
rm -r /data/registry
sudo ./install.sh
```

### Test the new harbor install (DNS has to be in place!)
```
https://orfdns.lab.lab

docker login orfdns.lab.lab  (admin/Vmware1)
docker tag nginx:latest orfdns.lab.lab/library/nginx:latest
docker push orfdns.lab.lab/library/nginx:latest
```

### Lifecycle / updates (incase you need to update cert or harbor yaml file)
```
https://goharbor.io/docs/1.10/install-config/reconfigure-manage-lifecycle/

sudo docker-compose down -v
sudo ./prepare
sudo ./prepare --with-notary --with-clair --with-chartmuseum
sudo docker-compose up -d
docker ps -a
```

### Get the cert for the cluster and generate TKG service configuration
```
openssl s_client -connect orfdns.lab.lab:443.  (and cut and paste into tag service configuration)
 Or
cat /usr/local/harbor/orfdns.lab.lab.cert | base64 -w0 > b2.txt
cat ~/orfdns.lab.lab.cert | base64 -w0 > b2.txt
cat b2.txt 

cat > b1.yaml <<-EOF
apiVersion: run.tanzu.vmware.com/v1alpha1
kind: TkgServiceConfiguration
metadata:
  name: tkg-service-configuration
spec:
  defaultCNI: antrea
  trust:
    additionalTrustedCAs:
      - name: harbor-vm-cert
        data: BASE64here
EOF

sed "s/BASE64here/`cat b2.txt`/" b1.yaml > tkgserviceconfiguration.yaml
```

### Get onto supervisor namespace (I have aliases for the login(s) and context swaps)
```
l1540
k1
```
### Apply the service configuration yaml and check out the before and after
```
k get tkgserviceconfiguration  -o yaml > a
kubectl apply -f ./tkgserviceconfiguration.yaml
k get tkgserviceconfiguration  -o yaml > b
sdiff a b
```

### Create a new guest cluster
```
k apply -f ~/7u2a/cluster.yaml   (https://github.com/ogelbric/Harbor_on_VM_and_Guestcluster_Cert/blob/main/cluster.yaml)
```

### Log into guest cluster
```
l2540
kubectl config use-context tkg-berlin
```
### Apply permissions
```
k apply -f ./authorize-psp-for-gc-service-accounts.yaml (https://github.com/ogelbric/Harbor_on_VM_and_Guestcluster_Cert/blob/main/authorize-psp-for-gc-service-accounts.yaml)
```
### Apply POD config with local harbor image regestry reference 
```
k apply -f ./nginx-local-harbor.yaml (https://github.com/ogelbric/Harbor_on_VM_and_Guestcluster_Cert/blob/main/nginx-local-harbor.yaml)
```
### Looking at evennts (k get events)
```
16s         Normal    Scheduled                  pod/nginx-77f5c65bb4-btdmz                       Successfully assigned default/nginx-77f5c65bb4-btdmz to tkg-berlin-workers-7ljst-8467947589-ftb7p
15s         Normal    Pulling                    pod/nginx-77f5c65bb4-btdmz                       Pulling image "orfdns.lab.lab/library/http-echo:latest"
14s         Normal    Pulled                     pod/nginx-77f5c65bb4-btdmz                       Successfully pulled image "orfdns.lab.lab/library/http-echo:latest"
14s         Normal    Created                    pod/nginx-77f5c65bb4-btdmz                       Created container nginx
13s         Normal    Started                    pod/nginx-77f5c65bb4-btdmz                       Started container nginx
16s         Normal    SuccessfulCreate           replicaset/nginx-77f5c65bb4                      Created pod: nginx-77f5c65bb4-64xqg
16s         Normal    SuccessfulCreate           replicaset/nginx-77f5c65bb4                      Created pod: nginx-77f5c65bb4-btdmz
16s         Normal    SuccessfulCreate           replicaset/nginx-77f5c65bb4                      Created pod: nginx-77f5c65bb4-bkmp5
11s         Normal    EnsuringLoadBalancer       service/nginx                                    Ensuring load balancer
16s         Normal    ScalingReplicaSet          deployment/nginx                                 Scaled up replica set nginx-77f5c65bb4 to 3
```

### Inspiration
```
https://tanzu.vmware.com/content/blog/how-to-set-up-harbor-registry-self-signed-certificates-tanzu-kubernetes-clusters
```


### Trouble shooting
```
        ssh to vCenter
	/usr/lib/vmware-wcp/decryptK8Pwd.py
	ssh to worker
         crictl pull orfdns.lab.local/library/nginx:latest


	export NAMESPACE=namespace1000
	
	kubectl -n $NAMESPACE get secret tkg-berlin-ssh -o jsonpath='{.data.ssh-privatekey}' | base64 -d > test-cluster-ssh-key
	chmod 600 test-cluster-ssh-key
	kubectl get secrets -A | grep $NAMESPACE | grep ssh

	kubectl -n $NAMESPACE get virtualmachines
	kubectl -n $NAMESPACE get virtualmachines | grep workers
	A=`kubectl -n $NAMESPACE get virtualmachines | grep workers | head -1 | awk '{print $1}'`
	VM_IP=`kubectl -n $NAMESPACE get virtualmachine $A  -o jsonpath='{.status.vmIp}'`

	#VM_IP=`kubectl -n $NAMESPACE get virtualmachine tkg-berlin-workers-2whdz-7c8b54bc7c-7wpds  -o jsonpath='{.status.vmIp}'`
	echo $VM_IP
	ssh -i test-cluster-ssh-key vmware-system-user@$VM_IP

	sudo crictl images
	sudo crictl pull orfdns.lab.local/library/nginx:latest
```



### Trouble shooting DNS on the worker node...
```
vmware-system-user@tkg-berlin-control-plane-bp62x [ ~ ]$ sudo crictl ps -a
CONTAINER           IMAGE               CREATED             STATE               NAME                           ATTEMPT             POD ID
bf18601782b83       44e90013d2071       5 days ago          Running             csi-resizer                    0                   49ac2e9f97fb5
d902851c29792       59d913ec2e044       5 days ago          Running             csi-provisioner                0                   49ac2e9f97fb5
f43ae7d5d2810       4d2e937854849       5 days ago          Running             liveness-probe                 0                   49ac2e9f97fb5
72f9b5d358f3c       dee637e9b719e       5 days ago          Running             vsphere-syncer                 0                   49ac2e9f97fb5
c822e726d9c2c       4d2e937854849       5 days ago          Running             liveness-probe                 0                   b0f63fb26bc73
f09c5cd108ecb       02e8179f74a07       5 days ago          Running             vsphere-csi-controller         0                   49ac2e9f97fb5
2ca513f7f8375       7841e1650cb15       5 days ago          Running             csi-attacher                   0                   49ac2e9f97fb5
c68570329573b       498104238f912       5 days ago          Running             guest-cluster-auth-service     0                   91be97b497427
d5fe2a66d8442       02e8179f74a07       5 days ago          Running             vsphere-csi-node               0                   b0f63fb26bc73
83f916e79130b       37cd9bd47067c       5 days ago          Running             coredns                        0                   7f8c98acfed25
2d96d71cb2122       37cd9bd47067c       5 days ago          Running             coredns                        0                   4705ddec81da2
7ce21c9d03aa1       fc69cbd2acfa1       5 days ago          Running             antrea-controller              0                   43b8bdb853631
c34d52988bebc       90bfa8d91a1d0       5 days ago          Running             guest-cluster-cloud-provider   0                   072dfba563cd8
cc5440b5e6320       9a3d9174ac1e7       5 days ago          Running             node-driver-registrar          0                   b0f63fb26bc73
8722f31ab8ce9       fc69cbd2acfa1       5 days ago          Running             antrea-ovs                     0                   964a52ea67877
9166f6c3c88c9       fc69cbd2acfa1       5 days ago          Running             antrea-agent                   0                   964a52ea67877
ae15213331139       fc69cbd2acfa1       5 days ago          Exited              install-cni                    0                   964a52ea67877
bff6639d7a621       babba6ff180ea       5 days ago          Running             docker-registry                0                   b5c289935ab68
abf8cf469f745       3dde81cc428db       5 days ago          Running             kube-proxy                     0                   1cdfc5dd0836a
9ed9cf02dbffe       4c8dfd2cd0c3c       5 days ago          Running             etcd                           0                   597267b6ca622
ecaf8da9860f4       7db6ffb163135       5 days ago          Running             kube-apiserver                 0                   b60a44fc4beca
4c9ec11115e8b       703d2edf85a3b       5 days ago          Running             kube-controller-manager        0                   aa2895c22325f
e56b592a7421b       ad3edad5d89bd       5 days ago          Running             kube-scheduler                 0                   9eca58a0c0800
vmware-system-user@tkg-berlin-control-plane-bp62x [ ~ ]$ sudo crictl logs --tail 100 83f916e79130b       
.:53
[INFO] plugin/reload: Running configuration MD5 = 4e235fcc3696966e76816bcd9034ebc7
CoreDNS-1.6.7
linux/amd64, go1.15.2, 
vmware-system-user@tkg-berlin-control-plane-bp62x [ ~ ]$ sudo crictl logs --tail 100 2d96d71cb2122       
.:53
[INFO] plugin/reload: Running configuration MD5 = 4e235fcc3696966e76816bcd9034ebc7
CoreDNS-1.6.7
linux/amd64, go1.15.2, 
vmware-system-user@tkg-berlin-control-plane-bp62x [ ~ ]$============================


sudo crictl exec -i -t   c68570329573b  /bin/bash
```

