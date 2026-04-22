# Web Solution with WordPress on AWS — Development Steps

## Architecture Overview

- **Web-Server**: RHEL 10, t3.micro, us-east-1a — Apache + PHP + WordPress
- **DB-Server**: RHEL 10, t3.micro, us-east-1a — MariaDB
- **Storage**: 3x EBS gp3 volumes (10 GiB each) attached to Web-Server, managed with LVM2
- **SSH Key**: `web-project-key.pem`

---

## STEP 0.1 — Create Web Server EC2 Instance

Launch an EC2 instance with the following configuration:

- **Name**: `Web-Server`
- **AMI**: Red Hat Enterprise Linux 10 (RHEL 10) — `ami-056244ee7f6e2feb8`
- **Instance type**: `t3.micro`
- **Region**: `us-east-1` (Norte de Virginia)
- **Key pair**: `web-project-key.pem`

---

## STEP 0.2 — Configure Web Server Security Group

Add inbound rules to the Web-Server security group:

| Type  | Protocol | Port | Source                 |
|-------|----------|------|------------------------|
| SSH   | TCP      | 22   | Custom: `79.116.147.59/32` |
| HTTP  | TCP      | 80   | Anywhere: `0.0.0.0/0` |

---

## STEP 0.3 — Create DB Server EC2 Instance

Launch a second EC2 instance:

- **Name**: `DB-Server`
- **AMI**: Red Hat Enterprise Linux 10 (RHEL 10) — `ami-056244ee7f6e2feb8`
- **Instance type**: `t3.micro`
- **Key pair**: `web-project-key.pem`

---

## STEP 0.4 — Configure DB Server Security Group

Add inbound rules to the DB-Server security group:

| Type          | Protocol | Port | Source                         |
|---------------|----------|------|--------------------------------|
| SSH           | TCP      | 22   | `0.0.0.0/0`                    |
| MySQL/Aurora  | TCP      | 3306 | Custom: `172.31.19.2/32` (Web-Server private IP) |

---

## STEP 0.5 — Create 3 EBS Volumes

Create three EBS volumes with the following settings:

- **Type**: gp3 (SSD)
- **Size**: 10 GiB each
- **IOPS**: 3000
- **Throughput**: 125 MiB/s
- **Availability Zone**: `us-east-1a`

Create a total of **3 volumes**.

---

## STEP 0.6 — Attach Volumes to Web-Server

Attach each volume to the Web-Server instance (`i-038fda81e442f286e`):

| Volume ID               | Device Name |
|-------------------------|-------------|
| `vol-07d785dae56f37858` | `/dev/sdb`  |
| `vol-000cd80676de40bd7` | `/dev/sdc`  |
| `vol-00512e84a3358475e` | `/dev/sdd`  |

---

## STEP 0.7 — SSH into Web-Server and Verify Attached Disks

```bash
ssh -i web-project-key.pem ec2-user@34.226.209.236
lsblk
```

**Expected output**: `nvme1n1`, `nvme2n1`, `nvme3n1` appear as 10G disks (no mount points yet).

---

## STEP 0.8 — Partition Disk 1 (nvme1n1)

```bash
sudo parted /dev/nvme1n1
(parted) mklabel gpt
(parted) mkpart primary 0% 100%
(parted) quit
```

---

## STEP 0.9 — Partition Disk 2 (nvme2n1)

```bash
sudo parted /dev/nvme2n1
(parted) mklabel gpt
(parted) mkpart primary 0% 100%
(parted) quit
```

---

## STEP 0.10 — Partition Disk 3 (nvme3n1)

```bash
sudo parted /dev/nvme3n1
(parted) mklabel gpt
(parted) mkpart primary 0% 100%
(parted) quit
```

---

## STEP 0.11 — Verify Partitions

```bash
lsblk
```

**Expected output**: `nvme1n1p1`, `nvme2n1p1`, `nvme3n1p1` created (10G each).

---

## STEP 0.12 — Install LVM2

```bash
sudo yum install lvm2 -y
```

> Note: If already installed, yum will report `lvm2-10:2.03.32-3.el10.x86_64` is already present.

---

## STEP 0.13 — Create Physical Volumes (PVs)

```bash
sudo pvcreate /dev/nvme1n1p1
sudo pvcreate /dev/nvme2n1p1
sudo pvcreate /dev/nvme3n1p1
sudo pvs
```

**Expected output**: 3 PVs of `<10.00g` each.

---

## STEP 0.14 — Create Volume Group

```bash
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```

This creates the Volume Group `webdata-vg` with ~29.99 GiB total.

---

## STEP 0.15 — Create Logical Volumes

```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
sudo lvs
```

**Expected output**:
- `apps-lv` — 14G in `webdata-vg`
- `logs-lv` — 14G in `webdata-vg`

---

## STEP 0.16 — Format Logical Volumes as EXT4

```bash
sudo mkfs.ext4 /dev/webdata-vg/apps-lv
sudo mkfs.ext4 /dev/webdata-vg/logs-lv
```

**UUIDs assigned**:
- `apps-lv`: `ecefa381-3036-45c4-9cec-2d5a10769d22`
- `logs-lv`: `136dbfa0-622d-4a59-a54c-d118f0d1f92f`

Verify the full LVM configuration:

```bash
sudo vgdisplay -v
```

---

## STEP 0.17 — Verify LVM Layout with lsblk

```bash
sudo lsblk
```

**Expected output**: `webdata--vg-apps--lv` (14G lvm) and `webdata--vg-logs--lv` (14G lvm) visible.

---

## STEP 0.18 — Create Mount Directories

```bash
sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs
```

---

## STEP 0.19 — Mount apps-lv and Backup Existing Logs

```bash
sudo mount /dev/webdata-vg/apps-lv /var/www/html
sudo rsync -av /var/log/ /home/recovery/logs/
```

---

## STEP 0.20 — Mount logs-lv and Restore Logs

```bash
sudo mount /dev/webdata-vg/logs-lv /var/log
sudo rsync -av /home/recovery/logs/ /var/log
```

---

## STEP 0.21 — Get UUIDs of Logical Volumes

```bash
sudo blkid
```

Note the UUIDs for `apps-lv` and `logs-lv` to use in `/etc/fstab`.

---

## STEP 0.22 — Edit /etc/fstab for Persistent Mounts

```bash
sudo vi /etc/fstab
```

Add the following lines:

```
UUID=ecefa381-3036-45c4-9cec-2d5a10769d22 /var/www/html ext4 defaults,nofail 0 2
UUID=136dbfa0-622d-4a59-a54c-d118f0d1f92f /var/log      ext4 defaults,nofail 0 2
```

---

## STEP 0.23 — Verify fstab and Apply Mounts

```bash
sudo cat /etc/fstab
sudo mount -a
```

---

## STEP 0.24 — Verify Mounts

```bash
df -h
```

**Expected output**:
```
/dev/mapper/webdata--vg-apps--lv  14G   24K  13G   1%  /var/www/html
/dev/mapper/webdata--vg-logs--lv  14G  1.1M  13G   1%  /var/log
```

---

## STEP 0.25 — Install MariaDB on DB-Server

SSH into the DB-Server and install MariaDB:

```bash
ssh -i web-project-key.pem ec2-user@<DB-Server-IP>
sudo yum install mariadb-server -y
```

---

## STEP 0.26 — Start and Enable MariaDB

```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

---

## STEP 0.27 — Check MariaDB Status

```bash
sudo systemctl status mariadb
```

**Expected output**: `active (running)` — MariaDB 10.11 database server.

---

## STEP 0.28 — Create WordPress Database

```bash
sudo mysql
```

Inside the MariaDB shell:

```sql
CREATE DATABASE wordpress;
```

---

## STEP 0.29 — Create DB User and Grant Permissions

Inside the MariaDB shell:

```sql
CREATE USER 'wpuser'@'%' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'%';
FLUSH PRIVILEGES;
```

Exit the MariaDB shell when done.

---

## STEP 0.30 — Install Apache (httpd) on Web-Server

Back on the Web-Server, install Apache HTTP Server:

```bash
sudo yum install httpd -y
```

Installed packages include: `httpd 2.4.63-4.el10_1.3`, `apr`, `apr-util`, `httpd-core`, `httpd-tools`, etc.

---

## STEP 0.31 — Start and Enable Apache

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
```

**Expected output**: `active (running)` — Apache HTTP Server listening on port 80.

---

## STEP 0.32 — Install PHP and PHP-MySQLnd

```bash
sudo yum install php php-mysqlnd -y
```

Installed packages include:
- `php 8.3.29-1.el10_1`
- `php-mysqlnd 8.3.29-1.el10_1`
- Dependencies: `php-common`, `php-pdo`, `php-cli`, `php-fpm`, `php-mbstring`, `php-opcache`, `php-xml`

---

## STEP 0.33 — Install wget and Download WordPress

```bash
sudo yum install wget -y
cd /tmp
wget https://wordpress.org/latest.tar.gz
```

WordPress archive (`latest.tar.gz`, ~26M) is downloaded to `/tmp`.

---

## STEP 0.34 — Extract WordPress and Copy to Web Root

```bash
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/
sudo chown -R apache:apache /var/www/html
sudo chmod -R 755 /var/www/html
```

---

## STEP 0.35 — Configure WordPress (wp-config.php)

Copy or rename `wp-config-sample.php` to `wp-config.php` and set the following values:

```php
define('DB_NAME',     'wordpress');
define('DB_USER',     'wpuser');
define('DB_PASSWORD', 'password123');
define('DB_HOST',     '54.211.94.220');   // DB-Server public IP
define('DB_CHARSET',  'utf8mb4');
```

---

## STEP 0.36 — Confirm Web Root, Permissions, and Restart Apache

```bash
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/
sudo chown -R apache:apache /var/www/html
sudo chmod -R 755 /var/www/html
sudo systemctl restart httpd
```

---

## STEP 0.37 — Open WordPress in the Browser

Navigate to the Web-Server public IP in your browser:

```
http://34.226.209.236/wp-admin/setup-config.php
```

You should see the **WordPress setup wizard** asking for:
1. Database name
2. Database username
3. Database password
4. Database host
5. Table prefix

Click **"Let's go!"** to complete the WordPress installation.

---

## STEP 0.38 — Click "Let's go!"

On the first WordPress setup screen, click the **"Let's go!"** button to open the database configuration form.

Expected result:
- WordPress opens `setup-config.php?step=1`
- A form appears with the fields for database name, username, password, database host, and table prefix

---

## STEP 0.39 — Fill in the Database Credentials

Complete the database connection form with the MariaDB details created earlier:

| Field | Value |
|-------|-------|
| Database Name | `wordpress` |
| Username | `wpuser` |
| Password | `password123` |
| Database Host | `172.31.21.155` |
| Table Prefix | `wp_` |

Then click **Submit**.

> Use the **DB-Server private IP** as the database host so the Web-Server connects over the internal AWS network.

---

## STEP 0.40 — Confirm WordPress Is Running

After WordPress finishes the setup, open the site in the browser:

```
http://34.226.209.236/
```

Expected result:
- The WordPress home page loads successfully
- The site title is displayed as **"Mi Proyecto WordPress"**
- The default **"Hello world!"** post is visible

---

## Summary

| Component    | Value                          |
|--------------|-------------------------------|
| Web-Server IP (public)  | `34.226.209.236`    |
| DB-Server IP (public)   | `54.211.94.220`     |
| Web-Server IP (private) | `172.31.22.35`      |
| DB-Server IP (private)  | `172.31.21.155`     |
| OS           | RHEL 10 (both servers)         |
| Web Server   | Apache httpd 2.4.63            |
| PHP          | 8.3.29                         |
| Database     | MariaDB 10.11                  |
| DB Name      | `wordpress`                    |
| DB User      | `wpuser`                       |
| DB Host      | `172.31.21.155`                |
| LVM VG       | `webdata-vg` (~29.99 GiB)      |
| apps-lv      | 14G → mounted at `/var/www/html` |
| logs-lv      | 14G → mounted at `/var/log`    |
| WordPress Status | Installed and reachable on port 80 |
