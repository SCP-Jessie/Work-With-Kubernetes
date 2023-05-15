# Work-With-Kubernetes
Short project and report demoing how to set up Docker, Kubernetes, and have a single-node nginx docker image running in Kubernetes with a setup allowing for easy multi-node configuration.

At the start of the project or every time a node reboots, ```sudo swapoff -a``` needs to be entered. This disables swapping of files which allows for Kubernetes' isolation features to work. 

Also enter ``bash```
# Install Docker Engine:
Ensures the necessary packages are present for Docker installation
```
sudo apt install apt-transport-https ca-certificates curl software-properties-common gnupg curl lsb-release
```

Add Docker's official GPG key
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Installation using repository
```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

Update the package index and install latest version of Docker CE
```
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Can verify if installation is successful with a hello world (optional):
```sudo docker run hello-world```

For people running Kubernetes 1.24 or later, cgroup driver must be changed as per Kubernetes requirements: 
"Docker Engine does not implement the CRI which is a requirement for a container runtime to work with Kubernetes. For that reason, an additional service cri-dockerd has to be installed. cri-dockerd is a project based on the legacy built-in Docker Engine support that was removed from the kubelet in version 1.24."
Therefore, make sure Docker uses systemd as cgroup driver. This configuration changes the cgroup driver
```
sudo bash -c 'cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF'
```

Create systemd directory for docker:
```sudo mkdir -p /etc/systemd/system/docker.service.d```

Restart Docker to apply changes from cgroup change:
```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Verifies that Docker is using systemd as cgroup driver (optional): 
```
sudo docker info | grep -i cgroup
```
Output should resemble:
```
Cgroup Driver: systemd
Cgroup Verion: 1
```

# Installing Go: https://golangdocs.com/install-go-linux 
Using Go is necessary to install cri-dockerd service on Ubuntu systems

```
sudo apt update
```

Get tarball. The version should not matter too much as it is just used once to build cri-dockerd
```
wget https://go.dev/dl/go1.20.3.linux-amd64.tar.gz
```
Extract tarball
```
sudo tar -C /usr/local/ -xzf go1.20.3.linux-amd64.tar.gz
```
Open the .profile file to set the PATH variable for Go as it is not set. Nano is uesd for this operation. This will ultimately allow Go commands to be run
```
sudo nano $HOME/.profile
```
Append text and save to bottom of file: 
```
export PATH=$PATH:/usr/local/go/bin
```
(Ctrl+O to exit, press Enter, then do Ctrl+X, press Enter to exit)

Apply changes to the system
```
source .profile
```

Command to check go is installed correctly (optional):
```
go version
```

# Install cri-dockerd service:
Clone the cri-dockerd repo and move into its directory
```
cd && git clone https://github.com/Mirantis/cri-dockerd.git
cd cri-dockerd
```
Build the code
```
sudo mkdir bin
go build -o ../bin/cri-dockerd
```
install cri-dockerd
```
cd .. && mkdir -p /usr/local/bin
sudo install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
```

Get services
```
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
```

Configure cri-dockerd to work with systemd
```
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
```
Start service with cri-dockerd enabled
```
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
```
Verify running status (optional):
```
sudo systemctl status cri-docker.socket
```
# Kubernetes installation:
Add Google gpg key so repository can be trusted
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
Set up repository
```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Update the package list and use apt-cache policy to inspect versions available in the repository
```
sudo apt-get update
apt-cache policy kubelet | head -n 20 
```
Install the required packages; if needed we can request a specific version by changing the VERSION variable 
```
VERSION=1.26.0-00
sudo apt-get install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION 
sudo apt-mark hold kubelet kubeadm kubectl
```
Ceck status of kubelet - should be in a crash loop (optional):
```
sudo systemctl status kubelet.service 
```
Ensure kubelet set to start when the system starts up
```
sudo systemctl enable kubelet.service
```
Inititalize Kubernetes control plane:
```
sudo kubeadm init --pod-network-cidr=<your_node_ip_addr>:///var/run/cri-dockerd.sock
```
For multi-node clusters, make sure to copy the full join command is displayed in the output upon successful initalization of the control plane. 

Eitherway, start the cluster using the commands given in the output:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
To easily manage the network of the pods, Calico is installed and applied. It is free and easy-to-use for simple setups like this.
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
```
Note that the cidr attribute in ipPools needs to be modified to whatever cidr your pod is using. Same as ip addr here. This file is hosted publicly on my repo and comes from Calico. 
```
kubectl create -f https://raw.githubusercontent.com/SCP-Jessie/Work-With-Kubernetes/main/custom-resources.yaml
```
According to Calico the ipPools cannot be changed after installation, so if a mistake is made somehow: 
```
kubectl delete -f https://raw.githubusercontent.com/<repo_path_of_mistake>/<messed_up_file_name>.yaml
```
And repeat the ```kubectl create -f``` step with the corrected file

Make sure that pods are running: May have to wait a couple moments for all statuses to update to running
```
watch kubectl get pods -n calico-system
```
# Create a Kubernetes deployment with Nginx:
```
kubectl create deployment nginx --image=nginx
```
See active deployments (optional):
```
kubeclt get deployments
```
Create service - A service deployment is necessary to access the nginx deployment
```
kubectl create service nodeport nginx --tcp=80:80
```
See services - extract the port number from the corresponding service:
```
kubectl get service
```

Finally, open a browser window and go to: <ip_addr>:<extracted_port_num>
![My Image](success.jpg)
Yay!

# Some Troubleshooting:
``` kubectl describe nodes ``` or ```kubectl describe node <node_name>``` can print out important information regarding a node's status and any issues it might have, like a lack of resources if the MemoryPressure, DiskPressure, or PIDPressure inder Conditions: Type have a status of True. The latter command will show the detailed descriptions only. 

# Links to all my resources and credits:
* https://kubernetes.io/docs/reference/kubectl/docker-cli-to-kubectl/
* https://prefetch.net/blog/2019/10/16/the-beginners-guide-to-creating-kubernetes-manifests/https://prefetch.net/blog/2019/10/16/the-beginners-guide-to-creating-kubernetes-manifests/
* https://app.pluralsight.com/library/courses/kubernetes-installation-configuration-fundamentals/table-of-contents
* https://app.pluralsight.com/library/courses/kubernetes-getting-started/table-of-contents
* https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
* https://www.mirantis.com/blog/how-to-install-cri-dockerd-and-migrate-nodes-from-dockershim/
* https://kubernetes.io/docs/setup/production-environment/container-runtimes/
* https://kubernetes.io/docs/reference/kubectl/cheatsheet/
* https://earthly.dev/blog/k8cluster-mnging-blding-kubeadm/

Calico:
* https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

Troubleshooting Guide:
* https://odsc.medium.com/common-issues-with-kubernetes-deployments-and-how-to-fix-them-dd3b949df87
