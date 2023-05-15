# Work-With-Kubernetes
Short project and report demoing how to set up Docker, Kubernetes, and have a single-node nginx docker image running in Kubernetes with a setup allowing for easy multi-node configuration.

At the start of the project or every time a node reboots, ```sudo swapoff -a``` needs to be entered. This disables swapping of files which allows for Kubernetes' isolation features to work. 

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


# Links to all my resources and credits:
* https://kubernetes.io/docs/reference/kubectl/docker-cli-to-kubectl/
* https://prefetch.net/blog/2019/10/16/the-beginners-guide-to-creating-kubernetes-manifests/https://prefetch.net/blog/2019/10/16/the-beginners-guide-to-creating-kubernetes-manifests/
* https://app.pluralsight.com/library/courses/kubernetes-installation-configuration-fundamentals/table-of-contents
* https://app.pluralsight.com/library/courses/kubernetes-getting-started/table-of-contents
* https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
