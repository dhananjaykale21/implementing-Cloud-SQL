# implementing-Cloud-SQL
Implementing Cloud SQL


Create a Cloud SQL database, 

Configure a virtual machine to run a proxy, 

Create a connection between an application and Cloud SQL, 

Connect an application to Cloud SQL using Private IP address

![image](https://github.com/user-attachments/assets/8138609d-2134-49b3-818e-fa0fa9a1242a)

==============================================================================
### Implementing Cloud SQL

#### Step 1: Enable Required APIs

Go to **Google Cloud Console** and enable the following APIs:

1. Cloud SQL Admin API
2. Compute Engine API
3. IAM API
4. VPC Network API

These APIs can be enabled via the **APIs & Services > Library** section.

---

#### Step 2: Create a Cloud SQL Instance

1. **Navigate to Cloud SQL:**
   - Go to **Navigation Menu > SQL**
   - Click **Create Instance**
   - Select **MySQL**
2. **Configure the instance:**
   - **Instance ID:** `wordpress-db`
   - **Root Password:** Set a secure password (store it securely for future use)
   - **Edition:** `Enterprise` (or Standard if Enterprise is unavailable in the free tier)
   - **Region:** `us-central1`
   - **Database Version:** `MySQL 5.7`
3. **Expand "Show Configuration Options"**
   - **Machine Configuration:** `1 vCPU, 3.75 GB RAM`
   - **Storage Type:** `SSD`, with a capacity of `10 GB`
   - **Connections:**
     - Enable **Private IP**
     - Select **default network**
     - Click **Set up Connection** > Enable API > Use an automatically allocated IP range > **Continue** > **Create Connection**
4. Click **Create Instance** and wait for it to be ready.

---

#### Step 3: Create Two Virtual Machines

Two VM instances need to be created:

1. `wordpress-proxy` (for external connection via proxy)
2. `wordpress-private-ip` (for Private IP connection to Cloud SQL)

##### **Creating the Virtual Machines**

1. **Go to Compute Engine > VM Instances**
2. **Click "Create Instance"**
   - **Name:** `wordpress-proxy`
   - **Region:** `us-central1`
   - **Machine Type:** `e2-micro` (free-tier eligible)
   - **Boot Disk:**
     - Click **Change**
     - Select **Debian 11** or **Ubuntu 22.04**
   - **Firewall:** Check `Allow HTTP traffic`
   - **Startup Script:** Enter the following script to install dependencies and set up WordPress:

```bash
#!/bin/bash
# Update system packages
apt-get update -y

# Install necessary dependencies for Apache, PHP, and MySQL client
apt-get -y install apache2 php libapache2-mod-php php-common php-mysql php-gmp php-curl php-intl php-mbstring php-xmlrpc php-gd php-xml php-cli php-zip default-mysql-client wget

# Download and set up the latest version of WordPress
wget https://wordpress.org/latest.tar.gz
 tar -zxf latest.tar.gz
 cd wordpress
 cp -r . /var/www/html
 cd /var/www/html
 rm index.html

# Set appropriate permissions for Apache
chgrp www-data .
chmod g+w .
```

3. Repeat the same process for `wordpress-private-ip`, ensuring it is in the **same region and VPC network** as the Cloud SQL instance.
4. For `wordpress-private-ip`, **do not assign an external IP** as it will be accessed via Private IP.
5. Click **Create** for both instances.

---

#### Step 4: Install Cloud SQL Proxy on `wordpress-proxy`

1. Open SSH for `wordpress-proxy` and execute:

```bash
sudo apt update && sudo apt install wget -y
wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
chmod +x cloud_sql_proxy
```

2. Retrieve the **Cloud SQL Instance Connection Name**:

   - Go to **Cloud SQL > wordpress-db > Overview**
   - Locate the **Instance Connection Name** (format: `project-id:region:instance-id`)

3. Start the Cloud SQL Proxy:

```bash
export SQL_CONNECTION="your-connection-name"
./cloud_sql_proxy -instances=$SQL_CONNECTION=tcp:3306 &
```

4. Verify the connection:

```bash
echo $SQL_CONNECTION
```

---

#### Step 5: Create and Configure Databases

1. **Go to Cloud SQL > wordpress-db > Databases**
2. Click **Create Database**
   - **Database Name:** `wordpress`
   - Click **Create**

---

#### Step 6: Connect WordPress to Cloud SQL

##### **For `wordpress-proxy` VM (via Cloud SQL Proxy)**

1. Retrieve the external IP of the VM:

```bash
curl -H "Metadata-Flavor: Google" http://169.254.169.254/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip
```

2. Open the IP in a browser and start the WordPress installation.
3. Enter the following details:
   - **Database Name:** `wordpress`
   - **Username:** `root`
   - **Password:** `[ROOT_PASSWORD]`
   - **Database Host:** `127.0.0.1`
   - Click **Submit > Run the Installation**

##### **For `wordpress-private-ip` VM (via Private IP)**

1. Retrieve the **Private IP of Cloud SQL** from **Cloud SQL > wordpress-db > Connections**.
2. Open the external IP of `wordpress-private-ip` in a browser.
3. Start the WordPress installation and enter:
   - **Database Name:** `wordpress`
   - **Username:** `root`
   - **Password:** `[ROOT_PASSWORD]`
   - **Database Host:** `[SQL_PRIVATE_IP]`
   - Click **Submit > Run the Installation**

---

### Final Steps: Test & Secure

✅ Test both WordPress setups via browser.
✅ Configure firewall rules to allow traffic.
✅ Ensure proper security settings for production use.






