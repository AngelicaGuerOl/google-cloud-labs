# Google Cloud Fundamentals: Cloud Storage and Cloud SQL

Hands-on Google Cloud lab in which a PHP web application was deployed on a Compute Engine VM, connected to a managed MySQL database through Cloud SQL, and integrated with an image stored in Cloud Storage.

## Objectives

- Deploy a web server on Compute Engine.
- Configure Apache and PHP using a startup script.
- Create a Cloud Storage bucket and upload an image.
- Configure public access to the image.
- Create and configure a Cloud SQL MySQL instance.
- Authorize the VM to connect to Cloud SQL.
- Connect the PHP application to MySQL using PDO.
- Display the Cloud Storage image on the web page.

---

## Architecture

```text
                        INTERNET
                            |
                            v
                  +-------------------+
                  |  Compute Engine   |
                  |      bloghost     |
                  |                   |
                  | Apache + PHP       |
                  |    index.php       |
                  +---------+---------+
                            |
                            | MySQL
                            v
                  +-------------------+
                  |     Cloud SQL     |
                  |      blog-db      |
                  |      MySQL 8.4    |
                  +-------------------+

                            |
                            | Image URL
                            v
                  +-------------------+
                  |  Cloud Storage    |
                  |      Bucket       |
                  | my-excellent-     |
                  | blog.png          |
                  +-------------------+
```

---

# 1. Compute Engine VM

A Compute Engine virtual machine was created to host the web application.

### Configuration

```text
Name:        bloghost
Zone:        europe-west4-a
Internal IP: 10.164.0.2
External IP: 34.7.255.95
```

HTTP traffic was enabled in the VM firewall configuration.

### Startup Script

The VM was automatically configured with Apache, PHP, and the PHP MySQL extension:

```bash
#!/bin/bash
apt-get install apache2 php php-mysql -y
service apache2 restart
```

### Concepts

**Compute Engine:** Google Cloud service used to run virtual machines.

**Virtual Machine (VM):** A virtualized computer used in this lab as the web server.

**Startup Script:** Script executed when the VM starts, allowing software and configuration to be installed automatically.

**Apache:** Web server responsible for serving the application.

**PHP:** Server-side language used to execute `index.php`.

### Evidence

![Compute Engine VM](screenshots/01-bloghost-vm.png)

---

# 2. Cloud Storage Bucket

A Cloud Storage bucket was created using Cloud Shell.

```bash
export LOCATION=US
gcloud storage buckets create -l $LOCATION gs://$DEVSHELL_PROJECT_ID
```

The bucket created during the lab was:

```text
qwiklabs-gcp-03-fc0b5da70879
```

### Configuration

```text
Location type: Multi-region
Location:      US
Storage class: Standard
```

### Concepts

**Cloud Storage:** Object storage service used to store files.

**Bucket:** Container where objects are stored.

**Object:** Individual file stored inside a bucket.

**Storage Class:** Defines the storage characteristics and access model for stored data.

### Evidence

![Cloud Storage Bucket](screenshots/02-cloud-storage-bucket.png)

---

# 3. Upload the Image

The image provided by the laboratory was downloaded from the training bucket:

```bash
gcloud storage cp gs://cloud-training/gcpfci/my-excellent-blog.png my-excellent-blog.png
```

The image was then uploaded to the new bucket:

```bash
gcloud storage cp my-excellent-blog.png gs://$DEVSHELL_PROJECT_ID/my-excellent-blog.png
```

The resulting object was:

```text
my-excellent-blog.png
```

### Evidence

![Cloud Storage Object Upload](screenshots/03-cloud-storage-object-upload.png)

---

# 4. Public Access to the Image

The object was configured to allow read access:

```bash
gsutil acl ch -u allUsers:R gs://$DEVSHELL_PROJECT_ID/my-excellent-blog.png
```

### Concept: ACL

An **Access Control List (ACL)** defines who can access a resource.

```text
allUsers:R
```

means:

```text
allUsers = any user
R         = read access
```

This allowed the image to be accessed through a public URL.

> `gsutil` displayed a recommendation to use `gcloud storage`, but the command completed successfully during the lab.

---

# 5. Cloud SQL Instance

A managed MySQL database was created using Cloud SQL.

### Configuration

```text
Instance:      blog-db
Database:      MySQL 8.4.10
Edition:       Enterprise
Machine:       2 vCPU, 8 GB
Region:        europe-west4
Availability:  Single zone
```

A database user was also created:

```text
Username: blogdbuser
```

### Concepts

**Cloud SQL:** Managed relational database service provided by Google Cloud.

**MySQL:** Relational database management system used by the application.

**Managed Service:** Google Cloud handles much of the underlying infrastructure and database administration.

### Evidence

![Cloud SQL Instance](screenshots/04-cloud-sql-instance.png)

---

# 6. Cloud SQL Networking

Cloud SQL was configured to use a **Public IP** connection.

The VM external IP was added as an authorized network:

```text
34.7.255.95/32
```

### Why `/32`?

```text
34.7.255.95/32
```

represents one specific IPv4 address.

Therefore, only that address was authorized instead of an entire IP range.

### Connection Flow

```text
bloghost
   |
   | External IP
   v
34.7.255.95
   |
   | Authorized Network
   v
Cloud SQL
```

### Evidence

![Cloud SQL Networking](screenshots/05-cloud-sql-network.png)

---

# 7. Configure the PHP Application

The Apache document root was accessed through SSH:

```bash
cd /var/www/html
sudo nano index.php
```

The application initially contained placeholders for the Cloud SQL address and password.

The database connection used PHP PDO:

```php
$dbserver = "CLOUDSQLIP";
$dbuser = "blogdbuser";
$dbpassword = "DBPASSWORD";

$conn = new PDO(
    "mysql:host=$dbserver;dbname=mysql",
    $dbuser,
    $dbpassword
);
```

### Concept: PDO

**PDO (PHP Data Objects)** is a PHP interface used to connect applications to databases.

In this lab, PDO was used to connect PHP with MySQL.

---

# 8. Database Connection Test

The first connection attempt failed because the placeholder:

```text
CLOUDSQLIP
```

had not yet been replaced with the actual Cloud SQL IP address.

The application displayed:

```text
Database connection failed: ...
```

### Evidence

![Database Connection Failed](screenshots/06-database-connection-failed.png)

### Error Analysis

The error occurred because PHP tried to resolve:

```text
CLOUDSQLIP
```

as a hostname.

Since it was only a placeholder and not a real database address, the connection could not be established.

---

# 9. Fix the Database Connection

The file was edited again:

```bash
sudo nano index.php
```

The placeholder was replaced with the real Cloud SQL public IP:

```php
$dbserver = "CLOUD_SQL_PUBLIC_IP";
```

The database user remained:

```php
$dbuser = "blogdbuser";
```

The configured database password was also inserted in the application.

After saving the file, Apache was restarted:

```bash
sudo service apache2 restart
```

The application then successfully connected to Cloud SQL.

### Result

```text
Connected successfully
```

### Evidence

![Database Connected](screenshots/07-database-connected.png)

---

# 10. Cloud Storage Integration

The public URL of:

```text
my-excellent-blog.png
```

was obtained from the Cloud Storage bucket.

The image was then referenced from `index.php` using HTML:

```html
<img src="PUBLIC_CLOUD_STORAGE_URL">
```

The working directory was:

```bash
cd /var/www/html
sudo nano index.php
```

After modifying the page, Apache was restarted:

```bash
sudo service apache2 restart
```

The final page displayed the image stored in Cloud Storage.

### Evidence

![Final Blog](screenshots/08-blog-with-cloud-storage-image.png)

---

# 11. Final Application Flow

```text
User
 |
 | HTTP
 v
Compute Engine
bloghost
 |
 +-- Apache
 |
 +-- PHP / index.php
 |      |
 |      +------> Cloud SQL
 |               MySQL
 |
 +------> Cloud Storage
          my-excellent-blog.png
```

The final application successfully demonstrated communication between compute, database, and storage services.

---

# 12. Main Commands Used

```bash
# Create the Cloud Storage bucket
export LOCATION=US
gcloud storage buckets create -l $LOCATION gs://$DEVSHELL_PROJECT_ID

# Download the image
gcloud storage cp gs://cloud-training/gcpfci/my-excellent-blog.png my-excellent-blog.png

# Upload the image
gcloud storage cp my-excellent-blog.png gs://$DEVSHELL_PROJECT_ID/my-excellent-blog.png

# Allow public read access
gsutil acl ch -u allUsers:R gs://$DEVSHELL_PROJECT_ID/my-excellent-blog.png

# Access the Apache document root
cd /var/www/html

# Edit the application
sudo nano index.php

# Restart Apache
sudo service apache2 restart
```

---

# 13. Technologies and Services

| Technology | Purpose |
|---|---|
| Compute Engine | Hosts the virtual machine |
| Apache | Web server |
| PHP | Application runtime |
| Cloud SQL | Managed database |
| MySQL | Relational database |
| Cloud Storage | Image storage |
| Cloud Shell | Google Cloud command-line environment |
| PDO | PHP database connectivity |

---

# 14. Key Concepts

### Compute Engine

Provides virtual machines that can be used to run applications and services.

### Cloud Storage

Provides object storage for files such as images, documents, backups, and other data.

### Cloud SQL

Provides managed relational databases such as MySQL.

### Public IP

An IP address that can be reached through the internet.

### Authorized Network

A Cloud SQL configuration that specifies which IP addresses are allowed to connect to an instance.

### CIDR `/32`

Represents a single IPv4 address.

### PDO

PHP interface used to establish database connections.

### Startup Script

Automates software installation and initial VM configuration.

---

# 15. Security Considerations

The configuration used in this lab was simplified for educational purposes.

In production environments, the following improvements would be recommended:

- Store credentials in **Secret Manager** instead of source code.
- Avoid exposing database errors to users.
- Prefer private connectivity when appropriate.
- Use encrypted database connections.
- Apply the principle of least privilege.
- Avoid public storage access unless required.
- Use a static IP when a stable external address is necessary.

---

# 16. Evidence Summary

```text
01-bloghost-vm.png
    Compute Engine VM

02-cloud-storage-bucket.png
    Cloud Storage bucket

03-cloud-storage-object-upload.png
    Image uploaded to Cloud Storage

04-cloud-sql-instance.png
    Cloud SQL MySQL instance

05-cloud-sql-network.png
    Authorized network configuration

06-database-connection-failed.png
    Initial database connection error

07-database-connected.png
    Successful database connection

08-blog-with-cloud-storage-image.png
    Final application with Cloud Storage image
```

---

# 17. Final Result

The lab successfully integrated the following Google Cloud services:

```text
+------------------+
| Compute Engine   |
|     bloghost     |
+--------+---------+
         |
         +--------------------+
         |                    |
         v                    v
+------------------+   +------------------+
|    Cloud SQL     |   |  Cloud Storage   |
|     blog-db      |   | my-excellent-    |
|     MySQL        |   | blog.png         |
+------------------+   +------------------+
```

The final application was able to:

- Run Apache and PHP on Compute Engine.
- Connect PHP to MySQL through Cloud SQL.
- Use an authorized external network for database connectivity.
- Store an image in Cloud Storage.
- Display the stored image on the web page.

## Conclusion

This laboratory provided practical experience integrating Google Cloud compute, database, networking, and storage services.

The most important troubleshooting step was correcting the database connection after the initial failure caused by the `CLOUDSQLIP` placeholder. Once the correct Cloud SQL IP address and credentials were configured, the application successfully connected to the database.

The final result demonstrated how a web application running on Compute Engine can communicate with Cloud SQL and consume static resources from Cloud Storage.