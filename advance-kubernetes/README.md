# HAProxy

`/etc/haproxy/haproxy.cfg`

```bash
foontend apiserver
	bind *:6443
	mode tcp
	option tcplog
	default_backend apiserver

backend apiserver
	option httpcheck GET /healthz
	http-check except status 200
	mode tcp
	option ssl-hello-chk
	balance roundrobin
	server master1 192.168.1.1 check
	server master2 192.168.1.2 check
	server master3 92.168.1.3 check
```

`/etc/keepalived/keepalived.conf`

```bash
vrrp_script_check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight 2
}

vrrp_instance VI_1 {
	state MASTER # BACKUP on ha2
	interface enp0s3
	virtual_router_id 1
	priority 100
	advert_int 5
	authentication {
		auth_type PASS
		auth_path mysecret
	}
	virtual_ipaddress {
    	192.168.1.20
    }
    track_script {
    	check_apiserver
    }
}
```

`/etc/keepalived/check_apiserver.sh`

```bash
#!/bin/bash

errorExit() {
	echo "*** $@" 1>&2
	exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error Get https://localhost:6443/"
if ip addr | grep -q "192.168.1.20";then
	curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://192.168.1.20:6443"
fi
```



```bash
chmod +x /etc/keepalived/check_apiserver.sh
apt install -y keepalived
systemctl enable keepalived
systemctl start keepalived
systemctl status keepalived
```

# install kubernetes

`master 1`

```bash
kubeadm init --controll-plane-endpoint="<virtual-ip>:6443" --apiserver-advertise-address="<master-node-ip>" --pod-network-cidr=10.10.0.0/16 --upload-certs 
```

```bash
# if you are regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
# if you are root
export KUBECONFIG=/etc/kubernetes/admin.conf
```

```bash
# you can join any number of the control-plane node
kubeadm join <virtual-ip>:6443 --token="<token>" --discovery-token-ca-cert-hash sha256:... --control-plane --certificate-key <certificate-key>
# you can join any number of worker nodes
kubeadm join <virtual-ip>:6443 --token="<token>" --discovery-token-ca-hash sha256:...
# delete node
kubectl delete node <node-name> # on master1
kubeadm reset # on master that we want to delete it.
```

## External etcd

```bash
# on master nodes
cat /etc/kubernetes/manifests/etcd.yaml
```

`github.com/etcd-io`

```bash
wget etcd
tar -zxvf etcd 
mv etcd* /usr/local/bin
NODE_IP="192.68.1.10"
ETCD_NAME=$(hostname -s)
ETCD1_IP="192.168.1.10"
ETCD2_IP="192.168.1.11"
ETCD3_IP="192.168.1.12"
```

```bash
cat <<EOF > /etc/systemd/system/etcd.service
[Unit]
Description=etcd

[Service]
Type=exec
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/pki/etcd.pem \\
  --key-file=/etc/etcd/pki/etcd-key.pem \\
  --peer-cert-file=/etc/etcd/pki/etcd.pem \\
  --peer-key-file=/etc/etcd/pki/etcd-key.pem \\
  --trusted-ca-file=/etc/etcd/pki/ca.pem \\
  --peer-trusted-ca-file=/tc/etcd/pki/ca.pem \\
  --client-cert-auth \\
  --data-dir=/var/lib/etcd \\
  --initial-advertise-peer-urls https://${ETCD_IP}:2380 \\
  --listen-peer-urls https://${ETCD_IP}:2380 \\
  --advertise-client-urls https://${NODE_IP}:2379 \\
  --listen-client-urls https://${NODE_IP}:2379,https://127.0.0.1:2379 \\
  --initial-cluster-token etcd-cluster-1 \\
  --initial-cluster etcd1=https://${ETCD1_IP}:2380,etcd2=https://${ETCD2_IP}:2380,etcd3=https://${ETCD3_IP}:2380 \\
  --initial-cluster-state new
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```bash
openssl req -newkey rsa:2048 -nodes -keyout <key-name> -x509 -days <expire-time> -out <certificate-name> 
mkdir -p /etc/etcd/pki
mv ca-key.pem ca.pem etcd.pem etcd-key.pem /etc/etcd/pki
```

```bash
systemctl daemon-reload
systemctl enable --now etcd.service
systemctl start etcd.service
```

```bash
etcdctl --endpoints=http://127.0.0.1:2379 put name hamid #OK without cert
etcdctl --endpoints=http://127.0.0.1:2379 get name hamid #hamid without cert
etcdctl --endpoints=https://127.0.0.1:2379 --cacert="/etc/etcd/pki/ca.pem" --cert="/etc/etcd/pki/etcd.pem" --key="/etc/etcd/pki/etcd-key.pem" member list # with cert
```

`kube-config.yaml`

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
controlPlaneEndPoint: <loadbalancer-ip>:6443      # multi-master
kubernetesVersion: v1.27.5 # multi-master
networking:
  podSubnet: "10.10.0.0/16"
etcd:
  external:
    endpoints:
      - https://192.168.1.10:2379
      - https://192.168.1.11:2379
      - https:// 192.168.1.12:2379
    caFile: /etc/kubernetes/pki/etcd/ca.pem
    certFile: /etc/kubernetes/pki/etcd/etcd.pem
    keyFile: /etc/kubernetes/pki/etcd/etcd-key.pem
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "<master-node-ip>"
```

```bash
kubeadm init --config kube-config.yaml --upload-cert

# generate new certs
kubeadm certs renew all

kubeadm token create --show-join-command # create new <token> and show worker join command
kubeadm init phase upload-certs --upload-certs # create new <certificate-key> 

kubeadm join <virtual-ip>:6443 --token="<token>" --discovery-token-ca-cert-hash sha256:... --control-plane --certificate-key <certificate-key>

kubectl get secret kubeadm-certs -n kube-system -o yaml
cat ca.pem | base64 -w 0

# cd /etc/cni/net.d
```

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# test kubernetes connection
etcdctl get / --endpoints=https://192.168.1.10:2379 --cacert=/etc/kubernetes/pki/etcd/ca.pem --cert=/etc/kubernetes/pki/etcd/etcd.pem --key=/etc/kubernetes/pki/etcd/etcd-key.pem --prefix --keys-only
```

## Service account

```bash
kubectl get sa -A
kubectl describe pod <pod-name>
# Mounts /var/run/secrets/kubernetes.io/serviceaccount
kubectl exex -it <pod-name> -- sh
ls /var/run/secrets/kubernetes.io/serviceaccount # ca.crt namespace token
cat token # this token has no access
curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -X GET https://kubernetes/api # 403

curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -X GET https://kubernetes/api -H "Authorization: Bearer <token >"

#show pods in namespace
curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -X GET https://kubernetes/api/v1/namespaces/default/pods -H "Authorization: Bearer <token >" # 403
```

`role.yaml`

```yaml
apiVersion: rbac.autorization.k8s.io/v1
kind: Role
metadata:
  name: bot
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list"]
```

`rolebinding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bot
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: bot
subjects:
  - apiGroup: ""
    kind: ServiceAccount
    name: <serviceaccount-name> # like default
    namespace: default
```

```bash
#show pods in namespace
# now we have access to get pods list
curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -X GET https://kubernetes/api/v1/namespaces/default/pods -H "Authorization: Bearer <token >" 
```

```bash
kubectl create serviceaccount <serviceaccount-name>
kubectl delete serviceaccount <serviceaccount-name>
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <serviceaccount-name>
  namespace: <namespace-name>
automountServiceAccountToken: true
```

