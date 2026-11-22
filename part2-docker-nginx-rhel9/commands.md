# Part 2 – Commands (RHEL 9 VM, IP 192.168.10.128)

## 1. Install Docker CE
```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
docker version          # Docker Engine – Community 28.1.1
```

## 2. HTTP container (image `static-website`)
```bash
cd ~/"Persora LLC"
docker build -t static-website .
docker run -d -p 8080:80 --name my-website static-website
# open http://localhost:8080
```

## 3. Self-signed SSL certificate (OpenSSL)
Standard OpenSSL command for a 1-year self-signed certificate (rebuilt; adjust to the command you used).
```bash
mkdir -p part2-docker-nginx-rhel9/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout part2-docker-nginx-rhel9/ssl/persora.key \
  -out    part2-docker-nginx-rhel9/ssl/persora.crt \
  -subj "/C=AE/O=Persora Lab/CN=192.168.10.128"
```

## 4. HTTPS container (image `persora-secure`)
```bash
docker build -f part2-docker-nginx-rhel9/Dockerfile.https -t persora-secure .
```

### Port conflict and container name conflict (what happened and the fix)
Apache (`httpd`) was already listening on port 80, and an earlier `persora-container` was still there in the `Created` state:
```bash
sudo systemctl stop httpd
sudo systemctl disable httpd        # stops Apache taking port 80 again after a reboot

docker run -d -p 80:80 -p 443:443 --name persora-container persora-secure
# Error: Conflict. The container name "/persora-container" is already in use

docker ps -a                        # shows the old container in the "Created" state
docker rm persora-container
docker run -d -p 80:80 -p 443:443 --name persora-container persora-secure
# open https://192.168.10.128  (the browser warns "Not secure" because the certificate is self-signed)
```

## 5. firewalld – custom zone `webzone`
```bash
sudo firewall-cmd --get-active-zones          # docker (docker0), public (ens160)
sudo firewall-cmd --permanent --new-zone=webzone
sudo firewall-cmd --permanent --zone=webzone --change-interface=ens160

sudo firewall-cmd --permanent --zone=webzone --add-service=http
sudo firewall-cmd --permanent --zone=webzone --add-service=https
sudo firewall-cmd --permanent --zone=webzone --add-port=8080/tcp   # HTTP test container
sudo firewall-cmd --permanent --zone=webzone --add-service=ssh     # remote administration
sudo firewall-cmd --reload

sudo firewall-cmd --zone=webzone --list-all
# services: http https ssh   ports: 8080/tcp
```
