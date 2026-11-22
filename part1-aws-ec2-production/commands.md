# Part 1 – Commands (AWS EC2, Amazon Linux 2023)

## 1. Starting point – ISPmanager hosting
The site was hosted on an ISPmanager VPS and managed with FileZilla (SFTP). I kept a full local copy of the site files (`C:\Users\...\Persora LLC\`) to move to AWS.

## 2. Launch EC2 and install Apache
- Instance: `t2.micro`, region us-east-1 (N. Virginia), Amazon Linux 2023
- Security Group inbound rules: 22/tcp (SSH, my IP only), 80/tcp and 443/tcp (anywhere)

```bash
sudo dnf install -y httpd mod_ssl
sudo systemctl enable --now httpd
```

## 3. Upload the site
I uploaded the site files with WinSCP (SFTP) to `/var/www/html/`, then set the owner and permissions:
```bash
sudo chown -R apache:apache /var/www/html
sudo find /var/www/html -type d -exec chmod 755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

## 4. DNS
In ISPmanager I pointed the domain's DNS **A records** for `persora.ae` and `www.persora.ae` to the EC2 public IP. I resolved an IPv6 conflict that was stopping Let's Encrypt from validating the domain: the domain must not resolve over IPv6 to a server that is not the EC2 instance. Then I waited for DNS propagation:
```bash
dig +short persora.ae
dig +short www.persora.ae
dig +short AAAA persora.ae   # check that no IPv6 record points somewhere else
```

## 5. Let's Encrypt SSL with Certbot
```bash
sudo dnf install -y certbot python3-certbot-apache
sudo certbot --apache -d persora.ae -d www.persora.ae
# Certificate:  /etc/letsencrypt/live/persora.ae/fullchain.pem
# Key:          /etc/letsencrypt/live/persora.ae/privkey.pem
# Deployed to   /etc/httpd/conf.d/persora-le-ssl.conf

sudo certbot renew --dry-run        # test automatic renewal
```
When I ran it again later, Certbot found the existing certificate. I chose **1 – reinstall** so that no new certificate was requested, which avoids Let's Encrypt rate limits.

## 6. HTTP → HTTPS redirect
See [`apache/persora.conf`](apache/persora.conf). Then:
```bash
sudo apachectl configtest && sudo systemctl reload httpd
```

## 7. Restart and failover check
I stopped and started the instance to check that Apache and HTTPS come back on their own after a reboot (`systemctl enable httpd`).
Note: without an **Elastic IP**, the EC2 public IPv4 address changes after a stop/start, and the DNS A records then have to be updated.
