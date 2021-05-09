# ansible-sample-cni

Ansible playbook written for easy distribution of [cni-from-scratch]'s `my-cni-demo`.
Please note that this only works for `ubuntu` distribution.
This will soon support `RHEL/CentOS` distribution.

## Preperations   

### Environments 
- All of the steps below have been executed with `root` privileges.
- Ensure the environment which no other CNI is installed, that is, just after executing only `kubeadm init` and `kubeadm join`.
- Check hostnames and addresses of all nodes in your cluster. For example, there are 3 nodes - 1 master, 2 workers.

### Passwordless ssh 
It is recommended that node running ansible can access others without a password.

For example, you can do it in the following ways:
```
# gen ssh pub key 
ssh-keygen -q -N "" 

# copy pub key to all hosts (including ansible host itself)
for host in cni-master01 cni-worker01 cni-worker02; do ssh-copy-id $host; done 
```
### Dependencies 
All nodes require `ansible` package to be installed. The remaining dependent packages are installed using `ansible`.
```
apt install -y ansible 
```

## Files 
These files were written for my test environment, so you need to rewrite it for your environment.

### inventory 

```
[all]
cni-master01
cni-worker01
cni-worker02
```
Add all hostnames in your cluster here.

### vars.yaml 

```
name: my-cni-demo
podcidr: 10.240.0.0/16
interface: ens32
bridge: cni0
nodeinfo:
- name: cni-master01
  ip: 192.168.100.100
  podcidr: 10.240.0.0/24
- name: cni-worker01
  ip: 192.168.100.101
  podcidr: 10.240.1.0/24
- name: cni-worker02
  ip: 192.168.100.102
  podcidr: 10.240.2.0/24
```
- name: Name of CNI binary 
- podcidr: Pod network CIDR for your cluster when `kubeadm init --pod-network-cidr <CIDR>` 
- interface: Host network interface for internet (ex. ethX, enpXXX, ensXX) 
- bridge: Virtual bridge name for container network 
- nodeinfo: 
  - name: hostname 
  - ip: ip address 
  - podcidr: Pod network CIDR for each node. Each node obtains IP CIDR for its own IPAM. (ex. /24 -> 254 pods per node)

## Installation

```
ansible-playbook deploy.yaml
```


## Result 

```
➜ cat << EOF | kubectl apply -f -                                                         
apiVersion: v1
kind: Pod
metadata:
  name: alpine1
spec:
  containers:
  - name: alpine
    image: alpine
    command:
      - "/bin/ash"
      - "-c"
      - "sleep 2000"
  nodeSelector:
    kubernetes.io/hostname: cni-master01
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx1
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    kubernetes.io/hostname: cni-master01
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx2
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    kubernetes.io/hostname: cni-worker01
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx3
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    kubernetes.io/hostname: cni-worker02
---
EOF
pod/alpine1 created
pod/nginx1 created
pod/nginx2 created
pod/nginx3 created

➜ kubectl get po -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
alpine1   1/1     Running   4          21h   10.240.0.6   cni-master01   <none>           <none>
nginx1    1/1     Running   0          21h   10.240.0.7   cni-master01   <none>           <none>
nginx2    1/1     Running   0          21h   10.240.1.2   cni-worker01   <none>           <none>
nginx3    1/1     Running   0          17h   10.240.2.2   cni-worker02   <none>           <none>

➜ kubectl exec -it nginx1 -- sh -c "echo nginx1 > /usr/share/nginx/html/index.html"
➜ kubectl exec -it nginx2 -- sh -c "echo nginx2 > /usr/share/nginx/html/index.html
➜ kubectl exec -it nginx3 -- sh -c "echo nginx3 > /usr/share/nginx/html/index.html"

➜ kubectl exec alpine1 -- wget -qO- 10.240.0.7 
nginx1

➜ kubectl exec alpine1 -- wget -qO- 10.240.1.2
nginx2

➜ kubectl exec alpine1 -- wget -qO- 10.240.2.2
nginx3
```

## Credits
Thanks to [@eranyanay]. His CNI is very intuitive and will be a very good guide for anyone who wants to start developing CNI. 

[cni-from-scratch]: https://github.com/eranyanay/cni-from-scratch
[@eranyanay]: https://github.com/eranyanay
