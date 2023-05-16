# How to do everything:
My original process for installing Docker and running an nginx image on Docker

1. Uninstall any pre-existing versions of Docker 

```
sudo apt-get remove docker docker-engine docker.io containerd runc 
``` 

2. Installation using repository

```
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
3. Add Docker's official GPG key
```
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

4. Enter bash
```
bash
```

5. Set up the repository - output of first command is placed into directory specifies by second ```sudo tee``` command.
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
6. Exit bash, then update
```
exit

sudo apt-get update
```

7. Installs latest version of Docker
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

8. Get nginx image
```
sudo docker pull nginx
```

9. Run image
```
sudo docker run -d --name server -p 80:80 nginx
```
