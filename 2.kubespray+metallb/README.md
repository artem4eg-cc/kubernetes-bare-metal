### HAProxy
```bash
root@haproxy-server1:~# apt -y install keepalived

root@haproxy-server1:/etc/haproxy# cat haproxy.cfg 

# add
listen kubernetes-apiserver-https
  bind *:8383
  mode tcp
  option log-health-checks
  timeout client 3h
  timeout server 3h
  server master1 10.128.0.103:6443 check check-ssl verify none inter 10000
  server master2 10.128.0.120:6443 check check-ssl verify none inter 10000
  server master3 10.128.0.130:6443 check check-ssl verify none inter 10000
  balance roundrobin
```
### kubespray
```bash
student@notebook:~/kubespray$ cp -rfp inventory/sample inventory/prod

student@notebook:~/kubespray/inventory/prod$ cat inventory.ini 

[all]
node1 ansible_host=10.128.0.103  
node2 ansible_host=10.128.0.120  
node3 ansible_host=10.128.0.130  
node4 ansible_host=10.128.0.28   
node5 ansible_host=10.128.0.111  

[kube_control_plane]
node1
node2
node3

[etcd]
node1
node2
node3

[kube_node]
# node2
# node3
node4
node5
# node6

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```
```bash
student@notebook:~/kubespray/inventory/prod$ nano group_vars/k8s_cluster/addons.yml

# Helm deployment
helm_enabled: true

# Metrics Server deployment
metrics_server_enabled: true
```
```bash
student@notebook:~/kubespray/inventory/prod$ nano group_vars/all/all.yml 

## External LB example config
apiserver_loadbalancer_domain_name: "lb.artem4eg.cc"
loadbalancer_apiserver:
  address: 51.250.65.14
  port: 8383
```

```bash
student@notebook:~/kubespray$ sudo apt update

student@notebook:~/kubespray$ sudo apt install sshpass

student@notebook:~/kubespray$ python3 -m venv venv

student@notebook:~/kubespray$ source venv/bin/activate

(venv) student@notebook:~/kubespray$ python3 -m pip install --upgrade pip

(venv) student@notebook:~/kubespray$ python3.10 -m pip install -U -r requirements.txt 

student@notebook:~/kubespray$ ansible-playbook -i inventory/prod/inventory.ini  --become -k --become-user=root -u student cluster.yml
```
### Metallb
```bash
kubectl edit configmap -n kube-system kube-proxy

apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```
```bash
cat <<EOF | kubectl apply -f - 
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.99.99.0/24
EOF
```
```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
EOF
```
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace

q@android-1:~/devops_projects/helm$ k get svc -n ingress-nginx 
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.233.45.118   10.99.99.0    80:31656/TCP,443:31186/TCP   86s
ingress-nginx-controller-admission   ClusterIP      10.233.15.112   <none>        443/TCP                      86s
```
```bash
# add in haproxy.cfg
frontend http_ingress
    bind *:80
    mode tcp
    default_backend ingress_nginx_http

frontend https_ingress
    bind *:443
    mode tcp
    default_backend ingress_nginx_https

backend ingress_nginx_http
    mode tcp
    maxconn 20000
    balance roundrobin
    server node4 10.128.0.28:31656 check send-proxy
    server node5 10.128.0.111:31656 check send-proxy
backend ingress_nginx_https
    mode tcp
    maxconn 20000
    balance roundrobin
    server node4 10.128.0.28:31186 check send-proxy
    server node5 10.128.0.111:31186 check send-proxy
```