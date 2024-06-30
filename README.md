    # Setting up K8s v1.23.17 on minikube

### Pre-requisite:

* System should have container runtime, here I used containerd
    * Did not use docker because of version compatibility issue
* Keep zypper updated and refresh it

### Installing containerd
1. `swapoff -a`
2. `sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab`

3. ```sh
    tee /etc/modules-load.d/containerd.conf <<EOF
    overlay
    br_netfilter
    EOF
    ```
4. `modprobe overlay`
5. `modprobe br_netfilter`
6. ```sh
    tee /etc/sysctl.d/kubernetes.conf<<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF
    ```
7. `sysctl --system`
8. `zypper refresh`
9. `zypper in containerd`
10. ```sh
    mkdir -p /etc/containerd
    containerd config default>/etc/containerd/config.toml
    sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
    ```
11. ```sh
    systemctl daemon-reload
    systemctl enable containerd
    systemctl start containerd
    systemctl status containerd
    ```
#### Install important repos
1. `zypper install conntrack-tools`
2. `zypper install iptables`
3. `zypper install docker`

### Set-up proxy
1. `vim /etc/sysconfig/proxy` to make sure the following are set
```
HTTP_PROXY=http://hpeproxy.its.hpecorp.net:80
HTTPS_PROXY=http://hpeproxy.its.hpecorp.net:443
NO_PROXY=10.103.0.0/16,10.96.0.0/16,192.168.0.0/16,localhost,127.0.0.1,us.cray.com,americas.cray.com,dev.cray.com,hpc.amslabs.hpecorp.net,eag.rdlabs.hpecorp.net,github.hpe.com,jira-pro.its.hpecorp.net
```
2. Re-login to shell
3. `vim /usr/lib/systemd/system/containerd.service` to edit and add under `[Service]` the following:
```
Environment="HTTP_PROXY=http://hpeproxy.its.hpecorp.net:80"
Environment="HTTPS_PROXY=http://hpeproxy.its.hpecorp.net:443"
Environment="NO_PROXY=10.103.0.0/16,10.96.0.0/16,192.168.0.0/16,localhost,127.0.0.1,us.cray.com,americas.cray.com,dev.cray.com,hpc.amslabs.hpecorp.net,eag.rdlabs.hpecorp.net,github.hpe.com,jira-pro.its.hpecorp.net"
```
4. `systemctl daemon-reload`
5. `systemctl restart containerd`

### Installing minikube
1. Download and install minikube latest version.
```sh
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
    sudo rpm -Uvh minikube-latest.x86_64.rpm
```
2. Start minikube with the required version(here v1.23.17)<br>
`minikube start --kubernetes-version=v1.23.17 --container-runtime=containerd --driver=none`
    * In case of errors faced try resolving them from Google
    * Using container-runtime as containerd, and driver as none.
        * Used driver as none because if I chose docker as driver it give error due to root privellages of docker.
    * If setting up gets error due to crictl, resolve using following:<br>
        `wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.25.0/crictl-v1.25.0-linux-amd64.tar.gz`<br>
        `sudo tar zxvf crictl-v1.25.0-linux-amd64.tar.gz -C /usr/local/bin`
        * Incase you still find crictl not found then try: `sudo tar zxvf crictl-v1.25.0-linux-amd64.tar.gz -C /usr//bin`
    * In case there is error that kubectl not found:<br>
        `curl -LO https://dl.k8s.io/release/v1.23.17/bin/linux/amd64/kubectl` <br>
        `sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl`
    * If required do: `systemctl enable kubelet`

3. Run `kubectl get node` to check if cluster is set-up.<br>
4. Run `kubectl get all` to see what all is running on the cluster.
5. Incase of error of bridge fail try this
    * To install a CNI plugin tgz in /opt/cni/bin, you can use the following steps:
    * Create a directory for the CNI: `mkdir -p /opt/cni/bin`
    * `Download the plugin: curl -O -L https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz`
    * Extract the CNI binary and move it to the directory: `tar -C /opt/cni/bin -xzf cni-plugins-linux-amd64-v0.9.1.tgz`


### Refrence:
* For Cri-dockerd:  https://www.mirantis.com/blog/how-to-install-cri-dockerd-and-migrate-nodes-from-dockershim




### Apply proxy in docker:
```sh
{
 "proxies": {
   "default": {
     "httpProxy": "http://hpeproxy.its.hpecorp.net:80",
     "httpsProxy": "http://hpeproxy.its.hpecorp.net:443",
     "noProxy": "10.103.0.0/16,10.96.0.0/16,192.168.0.0/16,localhost,127.0.0.1,us.cray.com,americas.cray.com,dev.cray.com,hpc.amslabs.hpecorp.net,eag.rdlabs.hpecorp.net,github.hpe.com,jira-pro.its.hpecorp.net"
   }
 }
}
```
