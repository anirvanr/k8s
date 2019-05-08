We will be setting up a Kubernetes cluster that will consist of one master and two worker nodes. All the nodes will run Ubuntu Xenial 64-bit OS.

Here is my full Vagrantfile gist. Run vagrant up to bring up a 3 node.

```bash
# -*- mode: ruby -*-
# vi: set ft=ruby :
IMAGE_NAME = "ubuntu/xenial64"
N = 2

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false

    config.vm.provider "virtualbox" do |v|
        v.memory = 2048
        v.cpus = 2
    end

    config.vm.define "k8s-master" do |master|
        master.vm.box = IMAGE_NAME
        master.vm.network "private_network", ip: "192.168.50.10"
        master.vm.hostname = "k8s-master"
    end

    (1..N).each do |i|
        config.vm.define "node-#{i}" do |node|
            node.vm.box = IMAGE_NAME
            node.vm.network "private_network", ip: "192.168.50.#{i + 10}"
            node.vm.hostname = "node-#{i}"
        end
    end
end
```

Setup Master

```
vagrant ssh k8s-master
vagrant@k8s-master:~$
vagrant@k8s-master:~$ sudo apt-get update
vagrant@k8s-master:~$ sudo apt-get install docker.io -y
vagrant@k8s-master:~$ sudo systemctl enable docker && sudo systemctl start docker
vagrant@k8s-master:~$ sudo apt-get install apt-transport-https curl -y
vagrant@k8s-master:~$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
vagrant@k8s-master:~$ sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
vagrant@k8s-master:~$ sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
vagrant@k8s-master:~$ sudo systemctl enable kubelet && sudo systemctl start kubelet
vagrant@k8s-master:~$ sudo swapoff -a
vagrant@k8s-master:~$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
vagrant@k8s-master:~$ sudo -i
root@k8s-master:~# IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`
root@k8s-master:~# kubeadm init --apiserver-advertise-address=$IP_ADDR --pod-network-cidr=10.244.0.0/16

....
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.50.10:6443 --token xg0vab.oiy36vp31elpixdx \
    --discovery-token-ca-cert-hash sha256:358af7ecee694b1df70f088c6bce6d3f0dcda61380a057bb995ced7c7d728f78
....

root@k8s-master:~# exit
vagrant@k8s-master:~$ mkdir -p $HOME/.kube
vagrant@k8s-master:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
vagrant@k8s-master:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
vagrant@k8s-master:~$ sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
vagrant@k8s-master:~$ kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
vagrant@k8s-master:~$ kubectl get node
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   13m   v1.14.1
vagrant@k8s-master:~$ kubectl get pod --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-fb8b8dccf-6b98l              1/1     Running   0          13m
kube-system   coredns-fb8b8dccf-mzt4g              1/1     Running   0          13m
kube-system   etcd-k8s-master                      1/1     Running   0          12m
kube-system   kube-apiserver-k8s-master            1/1     Running   0          12m
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          12m
kube-system   kube-flannel-ds-amd64-2w9gr          1/1     Running   0          109s
kube-system   kube-proxy-8r8cc                     1/1     Running   0          13m
kube-system   kube-scheduler-k8s-master            1/1     Running   0          12m
```

Setup nodes (all)

```
vagrant ssh node-1
vagrant@node-1:~$ sudo apt-get update
vagrant@node-1:~$ sudo apt-get install docker.io -y
vagrant@node-1:~$ sudo systemctl enable docker && sudo systemctl start docker
vagrant@node-1:~$ sudo apt-get install apt-transport-https curl -y
vagrant@node-1:~$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
vagrant@node-1:~$ sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
vagrant@node-1:~$ sudo apt-get update && sudo apt-get install kubeadm -y
vagrant@node-1:~$ sudo swapoff -a
vagrant@node-1:~$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
vagrant@node-1:~$ sudo kubeadm join 192.168.50.10:6443 --token xg0vab.oiy36vp31elpixdx \
    --discovery-token-ca-cert-hash sha256:358af7ecee694b1df70f088c6bce6d3f0dcda61380a057bb995ced7c7d728f78
```

https://github.com/kubernetes/kubernetes/issues/60835

"kubectl exec..", I get:

kubectl exec -it ...... -- bash

`error: unable to upgrade connection: pod does not exist`

This happens in Vagrant if you're running a multi-box setup because the kubelet on the workers end up binding services to the wrong ethernet interface. This error is the master attempting a connection to the wrong address in order to pull logs. The fix is to modify the config for the kubelet, as described in this blog post: https://medium.com/@joatmon08/playing-with-kubeadm-in-vagrant-machines-part-2-bac431095706

```
vagrant@node-1:~$ sudo vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
Environment="KUBELET_EXTRA_ARGS=--node-ip=192.168.50.11" # for node-2 node-ip=192.168.50.12

vagrant@node-1:~$ sudo systemctl daemon-reload
vagrant@node-1:~$ sudo systemctl stop kubelet && sudo systemctl start kubelet
```

Now go to master node and run below command to check master and slave node status

```
vagrant@k8s-master:~$ kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   79m   v1.14.1
node-1       Ready    <none>   48m   v1.14.1
node-2       Ready    <none>   44s   v1.14.1
```

Simple Smoke Test


```
vagrant@k8s-master:~$ kubectl run example --image=k8s.gcr.io/echoserver:1.10 --port=8080
vagrant@k8s-master:~$ kubectl expose deployment example --type=NodePort
vagrant@k8s-master:~$ kubectl get service
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
example      NodePort    10.105.34.67   <none>        8080:31406/TCP   23m
vagrant@k8s-master:~$ kubectl get pod -o wide
NAME                       READY   STATUS    RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
example-66c89bfbb6-rlqbl   1/1     Running   0          5m46s   10.244.2.3   node-2   <none>           <none>
```

Open `http://<node-2-ip>:31406` in browser


Drain node

```
vagrant@k8s-master:~$ kubectl drain node-2 --ignore-daemonsets
vagrant@k8s-master:~$ kubectl get node
NAME         STATUS                     ROLES    AGE    VERSION
k8s-master   Ready                      master   148m   v1.14.1
node-1       Ready                      <none>   117m   v1.14.1
node-2       Ready,SchedulingDisabled   <none>   69m    v1.14.1
```

vagrant halt node-2

```
vagrant@k8s-master:~$ kubectl get nodes
NAME         STATUS                     ROLES    AGE    VERSION
k8s-master   Ready                      master   153m   v1.14.1
node-1       Ready                      <none>   122m   v1.14.1
node-2       Ready,SchedulingDisabled   <none>   74m    v1.14.1
```

Mark node as schedulable again

```
vagrant@k8s-master:~$ kubectl uncordon node-2
```
