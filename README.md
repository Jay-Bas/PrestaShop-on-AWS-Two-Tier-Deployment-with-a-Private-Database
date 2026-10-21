# PrestaShop on AWS: Two-Tier Deployment with a Private Database

Deploying the open-source e-commerce platform **PrestaShop 9.2.0** on AWS using only free-tier resources, with the application and the database on **separate servers** and the data tier locked down to the web server alone.

> Built as a cybersecurity internship assessment. The environment was temporary and has been torn down, so no live URL is included.

## Architecture

```
                 Internet
                    |
          HTTP 80 / HTTPS 443 (open)
                    |
        +-----------------------+
        |  EC2 prestashop-web   |   SSH 22: my IP only (/32)
        |  Ubuntu 24.04, t3.micro|
        |  Apache 2.4 + PHP 8.3 |
        |  security group: web-sg|
        +-----------+-----------+
                    |
             MySQL 3306 (only from web-sg)
                    |
        +-----------+-----------+
        |  RDS prestashop-db    |   no public IP
        |  MySQL 8.4, encrypted |
        |  security group: db-sg|
        +-----------------------+
```

| Component | Detail |
|---|---|
| Region | eu-north-1 (Stockholm), default VPC |
| App server | EC2 t3.micro, Ubuntu Server 24.04 LTS, 20 GiB gp3 |
| Database | RDS MySQL 8.4, Single-AZ, 20 GiB gp3, encrypted with KMS |
| Cost control | Zero-spend budget alert, free-tier resources only |

## Security decisions

- **Database is private.** Public access is off, and `db-sg` allows port 3306 only from the `web-sg` security group (not an IP range), so access survives IP changes and nothing else can reach it.
- **SSH restricted to one IP** (`/32`) instead of `0.0.0.0/0`.
- **Removed a risky default.** The launch wizard opened port 1433 (MSSQL) to the internet; I replaced it with the SSH rule above.
- **Key rotation.** A private key was exposed before launch, so I generated a replacement and deleted the original.
- **Encryption at rest** on the database, with automated backups.
- **PrestaShop hardening:** deleted the `install` folder after setup; the back office folder uses the installer's random name.
- **Patching:** applied 190 pending updates (153 security) and rebooted to load the new kernel.

## Build steps

1. **Cost guardrail:** AWS Budgets, zero-spend template.
2. **EC2:** Ubuntu 24.04, t3.micro, key pair, security group `web-sg` (SSH from my IP, HTTP, HTTPS).
3. **RDS:** MySQL, free tier, private, initial database `prestashop`, new group `db-sg`.
4. **Connect:** harden the key file permissions on Windows, then SSH in.

   ```powershell
   icacls .\key.pem /inheritance:r          # drop inherited permissions
   icacls .\key.pem /grant:r "$($env:USERNAME):(R)"   # read-only for me
   ssh -i .\key.pem ubuntu@<public-ip>
   ```
5. **Verify the database path from the web server:**

   ```bash
   mysql -h <rds-endpoint> -u admin -p prestashop
   ```
6. **Web stack:**

   ```bash
   sudo apt update
   sudo apt install -y apache2 php libapache2-mod-php php-mysql php-curl php-gd \
     php-intl php-mbstring php-xml php-zip php-bcmath unzip mysql-client
   sudo systemctl enable --now apache2
   ```
7. **PrestaShop:** upload the official installer package, extract it into `/var/www/html`, give Apache ownership, enable `mod_rewrite`, raise PHP `memory_limit` to 512M, run the web installer, then delete `install/`.

   ```bash
   sudo unzip -q /tmp/ps/prestashop.zip -d /var/www/html
   sudo chown -R www-data:www-data /var/www/html
   sudo a2enmod rewrite
   sudo rm -rf /var/www/html/install
   ```

## Problems I hit and fixed

| Problem | Cause | Fix |
|---|---|---|
| Launch failed: SQL Server not supported for t3.micro | AMI was an Ubuntu image bundled with SQL Server | Used the plain Canonical Ubuntu AMI |
| RDS could not auto-link to EC2 (CIDR conflict) | Overlapping auto-created subnet | Created `db-sg` manually with `web-sg` as the source |
| SSH/SCP timed out | My public IP had changed | Re-set the SSH rule to My IP |
| No installer on GitHub releases | Releases ship source code only | Downloaded the official package from the PrestaShop site |
| Wrong engine preselected | Console defaulted to Aurora PostgreSQL | Chose MySQL with Full configuration |

## Screenshots

Sensitive values (account number, email, admin path, tokens, passwords) are redacted.

| | |
|---|---|
| ![Assignment brief](images/00-assignment-brief.jpg) | ![AWS sign-up](images/01-aws-signup.jpg) |
| ![RDS encryption and backup](images/05-rds-encryption-backup.jpg) | ![Installer: database connected](images/12-installer-db-connected.jpg) |
| ![Updates and reboot](images/14-updates-reboot.jpg) | |

## What I would add for production

HTTPS with a domain and certificate, a load balancer or WAF in front of the web tier, private subnets for the database, credentials in AWS Secrets Manager, Multi-AZ with longer backups, and CloudWatch alarms.

## Cleanup

Terminate the EC2 instance, delete the RDS instance (no final snapshot), then remove `web-sg`, `db-sg` and the key pair so nothing keeps billing.

## Author

**John Segun A** | Aspiring SOC Analyst | Cisco Networking Academy certified
