# implementing-Cloud-SQL
Implementing Cloud SQL


Create a Cloud SQL database, 

Configure a virtual machine to run a proxy, 

Create a connection between an application and Cloud SQL, 

Connect an application to Cloud SQL using Private IP address

![image](https://github.com/user-attachments/assets/8138609d-2134-49b3-818e-fa0fa9a1242a)




Since this lab setup includes pre-configured VMs and a temporary project in Qwiklabs, you'll need to manually set up everything on your GCP free-tier account. Below is a step-by-step guide for setting up Cloud SQL and connecting it via both external proxy and Private IP.

---

### **Step 1: Enable Required APIs**
Go to **Google Cloud Console** and enable the following APIs:
1. Cloud SQL Admin API
2. Compute Engine API
3. IAM API
4. VPC Network API

You can enable them via the **APIs & Services > Library** section.

---

### **Step 2: Create a Cloud SQL Instance**
1. **Navigate to Cloud SQL:**  
   - Go to **Navigation Menu > SQL**  
   - Click **Create Instance**  
   - Choose **MySQL**
2. **Configure the instance:**
   - **Instance ID:** `wordpress-db`
   - **Root Password:** Set a strong password (Save this; you'll need it later)
   - **Choose a Cloud SQL Edition:** `Enterprise` (You can try Standard if Enterprise isn't available in free tier)
   - **Region:** Choose the same region where your VM will be created (to enable Private IP connection)
   - **Zone:** Any
   - **Database Version:** `MySQL 5.7`
3. **Expand "Show configuration options"**
   - **Machine Configuration:** Choose `1 vCPU, 3.75 GB RAM` (or smallest available)
   - **Storage:**
     - **Type:** `SSD`
     - **Capacity:** `10 GB`
   - **Connections:**
     - Enable **Private IP**
     - Under **Network**, select `default`
     - Click **Set up Connection** > Enable API > Use an automatically allocated IP range > **Continue** > **Create Connection**
4. Click **Create Instance** and wait for it to be ready.

---

### **Step 3: Create Two Virtual Machines**
You need to create two VMs:  
1. `wordpress-proxy` (for external connection via proxy)  
2. `wordpress-private-ip` (for direct Private IP connection)

#### **Create the First VM (wordpress-proxy)**
1. **Go to Compute Engine > VM Instances**  
2. **Click "Create Instance"**  
   - **Name:** `wordpress-proxy`
   - **Region:** Same as Cloud SQL instance
   - **Machine Type:** `e2-micro` (free-tier eligible)
   - **Boot Disk:**
     - Click **Change**
     - Select **Debian 11** or **Ubuntu 22.04**
   - **Firewall:** Check `Allow HTTP traffic`
   - **Click Create**

#### **Create the Second VM (wordpress-private-ip)**
1. **Repeat the steps above**, but name it `wordpress-private-ip`.
2. Make sure it’s in the **same region** and **VPC network** as your Cloud SQL instance.
3. **Do not enable external IP** (we will connect via Private IP).
4. Click **Create**.

---

### **Step 4: Install Cloud SQL Proxy on `wordpress-proxy` VM**
1. Open SSH for `wordpress-proxy`:
   ```
   sudo apt update && sudo apt install wget -y
   wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
   chmod +x cloud_sql_proxy
   ```

2. Get the Cloud SQL **Connection Name**:
   - Go to **Cloud SQL > wordpress-db > Overview**  
   - Find **Instance Connection Name** (format: `project-id:region:instance-id`)  

3. Run the Cloud SQL Proxy:
   ```
   export SQL_CONNECTION="your-connection-name"
   ./cloud_sql_proxy -instances=$SQL_CONNECTION=tcp:3306 &
   ```

4. **Verify connection**:
   ```
   netstat -an | grep 3306
   ```

---

### **Step 5: Install MySQL and Configure WordPress**
On both `wordpress-proxy` and `wordpress-private-ip`:

1. Open SSH and run:
   ```
   sudo apt update
   sudo apt install apache2 php php-mysql mariadb-client unzip -y
   ```

2. **Download WordPress**:
   ```
   wget https://wordpress.org/latest.zip
   unzip latest.zip
   sudo mv wordpress /var/www/html/
   sudo chown -R www-data:www-data /var/www/html/wordpress
   ```

3. **Configure Apache**:
   ```
   sudo systemctl restart apache2
   ```

---

### **Step 6: Connect WordPress to Cloud SQL**
#### **For `wordpress-proxy` VM (via Cloud SQL Proxy)**
1. Get your **VM's External IP**:
   ```
   curl -H "Metadata-Flavor: Google" http://169.254.169.254/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip
   ```
   Open this IP in a browser.

2. Start WordPress Setup:
   - Click **Let’s Go**
   - **Database Name:** `wordpress`
   - **Username:** `root`
   - **Password:** `[ROOT_PASSWORD]`
   - **Database Host:** `127.0.0.1`
   - Click **Submit > Run the Installation**

---

### **Step 7: Connect `wordpress-private-ip` VM via Private IP**
1. **Get Private IP of Cloud SQL**
   - Go to **Cloud SQL > wordpress-db > Connections**
   - Note the **Private IP address**

2. Open SSH for `wordpress-private-ip`:
   ```
   sudo nano /var/www/html/wordpress/wp-config.php
   ```
   Find:
   ```
   define('DB_HOST', '127.0.0.1');
   ```
   Change it to:
   ```
   define('DB_HOST', '[SQL_PRIVATE_IP]');
   ```

3. Restart Apache:
   ```
   sudo systemctl restart apache2
   ```

4. **Open `wordpress-private-ip` external IP in a browser** and follow the same WordPress setup.

---

### **Step 8: Security & Firewall Rules**
1. **Allow WordPress traffic**
   - Go to **VPC Network > Firewall Rules**
   - Click **Create Rule**
   - **Name:** `allow-wordpress`
   - **Network:** `default`
   - **Targets:** `All instances in the network`
   - **Source IP Ranges:** `0.0.0.0/0`
   - **Protocols and Ports:** `TCP: 80`
   - Click **Create**

2. **Allow MySQL connections (for Private IP)**
   - Click **Create Rule**
   - **Name:** `allow-mysql-private`
   - **Network:** `default`
   - **Targets:** `wordpress-private-ip`
   - **Source IP Ranges:** `10.0.0.0/8`
   - **Protocols and Ports:** `TCP: 3306`
   - Click **Create**

---

### **Final Step: Test Both Connections**
- Go to **`wordpress-proxy` external IP** in a browser and check WordPress.
- Go to **`wordpress-private-ip` external IP** in a browser and check WordPress.

---

### **Summary**
✅ **Created Cloud SQL instance with Private IP enabled**  
✅ **Configured Cloud SQL Proxy on `wordpress-proxy` VM**  
✅ **Connected WordPress via external proxy (secure tunnel)**  
✅ **Connected WordPress via Private IP (direct secure connection)**  

This setup replicates the Qwiklabs environment but works on your **own free-tier GCP account**. 🚀


