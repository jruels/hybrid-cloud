
## 🧰 Prerequisites

Before starting, ensure you have the following:

1. **Terraform** installed on your local machine:
   - **Windows/macOS/Linux**: Download and install from [Terraform Downloads](https://developer.hashicorp.com/terraform/install).

2. **AWS CLI** installed and configured:
   - Installation guide: [Installing the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
   - Configure with `aws configure` using your AWS credentials.

3. **Azure CLI** installed and configured:
   - Installation guide: [Install the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
   - Log in with `az login`.

4. **Docker** installed on your local machine:
   - **Windows/macOS**: Download and install from [Docker Desktop](https://www.docker.com/products/docker-desktop/).
   - **Linux**: Follow the official [Docker Engine installation guide](https://docs.docker.com/engine/install/) for your distribution.

5. **Python 3.8+** installed:
   - **Windows/macOS**: Download from [Python.org](https://www.python.org/downloads/).
   - **Linux**: Use your package manager, e.g., `sudo apt install python3`.

6. **An AWS account** with permissions to create RDS instances.

7. **An Azure account** with permissions to create Azure Cache for Redis instances.
   
8. **MySQL Client** installed (to connect to your RDS instance):  
   - **Debian/Ubuntu**:  
     ```bash
     sudo apt update && sudo apt install mysql-client -y
     ```  
   - **Red Hat/CentOS**:  
     ```bash
     sudo yum install mysql -y
     ```  
   - **macOS (Homebrew)**:  
     ```bash
     brew install mysql-client
     echo 'export PATH="/usr/local/opt/mysql-client/bin:$PATH"' >> ~/.bash_profile
     ```  
   - **Windows**:  
     NB: Run the following commands in administrator mode
     1. **Install Chocolatey** (if not already installed):  
        ```powershell
        Set-ExecutionPolicy Bypass -Scope Process -Force
        [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12
        iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
        ```  
     2. **Install MySQL client** via Chocolatey:  
        ```powershell
        choco install mysql -y
        ```
---

## 🏗️ Step-by-Step Guide

### 1. Set Up Project Directory

Create a new directory for your project and navigate into it:

```bash
mkdir hybrid_app
cd hybrid_app
```

## 🛠️ Infrastructure Setup with Terraform

We'll create two separate Terraform configurations: one for AWS RDS MySQL and another for Azure Cache for Redis.

### a. AWS RDS MySQL Configuration

1. **Create a directory for AWS Terraform configuration:**

   ```bash
   mkdir aws_rds
   cd aws_rds
   ```

2. **Create `main.tf` with the following content:**

   ```hcl
    provider "aws" {
        region = "us-east-1"
    }

    # pull your public IP at apply‑time
    data "http" "myip" {
        url = "http://checkip.amazonaws.com/"
    }

    output "default_vpc_id" {
        value = data.aws_vpc.default.id
    }

    data "aws_vpc" "default" {
        default = true
    }

    resource "aws_security_group" "mysql_sg" {
        # use only ASCII hyphens in names
        name        = "mysql-sg"
        description = "allow MySQL from my IP"
        vpc_id      = data.aws_vpc.default.id
    
        ingress {
        from_port   = 3306
        to_port     = 3306
        protocol    = "tcp"
        cidr_blocks = ["${trimspace(data.http.myip.body)}/32"]
        }
    
        egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
        }
    }
     resource "aws_db_instance" "mysql" {
         allocated_storage    = 20
         engine               = "mysql"
         engine_version       = "8.0"
         instance_class       = "db.t3.micro"

         # rename `name` → `db_name`
         db_name              = "webappdb"

         # optional: set a stable RDS instance identifier
         identifier           = "webappdb-instance"

         username             = "admin"
         password             = "yourpassword"
         parameter_group_name = "default.mysql8.0"
         publicly_accessible  = true
         skip_final_snapshot  = true
         vpc_security_group_ids  = [aws_security_group.mysql_sg.id]
         tags = {
         Name = "WebAppDB"
         }
     }

     output "rds_endpoint" {
     value = aws_db_instance.mysql.endpoint
     }
   ```
   Replace `"us-east-1"` with an active AWS region of your choice.
   Replace `"yourpassword"` with a secure password of your choice.

3. **Initialize and apply the Terraform configuration:**

   ```bash
   terraform init
   terraform apply
   ```

   Confirm the apply step when prompted.

4. **Note the RDS endpoint:**

   After the apply completes, Terraform will output the RDS endpoint. Save this value for later use. For example:

   ```
   rds_endpoint = "webappdb-instance.cwbmawgukf0s.us-east-1.rds.amazonaws.com:3306"
   ```

### b. Azure Cache for Redis Configuration

1. **Navigate back to the project root and create a directory for Azure Terraform configuration:**

   ```bash
   cd ..
   mkdir azure_redis
   cd azure_redis
   ```

2. **Create `main.tf` with the following content:**

Replace the azure subscription_id and tenant_id with values from your account. You can get these values by running the following command:

```bash
az account list
```

Below are the contents of `main.tf`:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 3.0"
    }
  }
}

provider "azurerm" {
  subscription_id = "replace_with_your_azure_subscription_id"
  tenant_id       = "replace_with_your_azure_tenant_id"
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "webapp-rg"
  location = "East US"
}

resource "random_integer" "suffix" {
  min = 10000
  max = 99999
}

resource "azurerm_redis_cache" "redis" {
  name                = "webappredis${random_integer.suffix.result}"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  capacity            = 0
  family              = "C"
  sku_name            = "Basic"
  minimum_tls_version = "1.2"
}

output "redis_hostname" {
  value = azurerm_redis_cache.redis.hostname
}

output "redis_primary_access_key" {
    value     = azurerm_redis_cache.redis.primary_access_key
    sensitive = true
}
   ```

1. **Initialize and apply the Terraform configuration:**

   ```bash
   az login
   terraform init
   terraform apply
   ```

   Choose your Azure account and subscription when prompted after running `az login`
   Confirm the apply step when prompted.

2. **Note the Redis hostname and access key:**

   After the apply completes, Terraform will output the Redis hostname and primary access key. Save these values for later use.

---

## 🐍 Flask Application Setup

1. **Navigate back to the project root and create a directory for the Flask application:**

   ```bash
   cd ..
   mkdir flask_app
   cd flask_app
   ```

2. **Create a virtual environment and activate it:**
  **Please make sure you have installed Python before completing this task**

   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows, use 'venv\Scripts\activate'
   ```

3. **Install required packages:**

   ```bash
   pip install flask pymysql redis
   ```

4. **Create `flask_app/app.py` with the following content:**
   
 

   ```python
   from flask import Flask, request
   import pymysql
   import redis
   import os

   app = Flask(__name__)

   # MySQL configuration
   db_host = os.environ.get('DB_HOST', 'localhost')
   db_user = os.environ.get('DB_USER', 'admin')
   db_password = os.environ.get('DB_PASSWORD', 'yourpassword')
   db_name = os.environ.get('DB_NAME', 'webappdb')

   # Redis configuration
   redis_host = os.environ.get('REDIS_HOST', 'localhost')
   redis_port = int(os.environ.get('REDIS_PORT', 6380))
   redis_password = os.environ.get('REDIS_PASSWORD', None)

   # Initialize Redis client
   r = redis.Redis(host=redis_host, port=redis_port, password=redis_password, ssl=True)

   @app.route('/')
   def index():
       # Connect to MySQL
       try:
           conn = pymysql.connect(host=db_host, user=db_user, password=db_password, database=db_name)
           cursor = conn.cursor()
           cursor.execute("SELECT COUNT(*) FROM visits")
           count = cursor.fetchone()[0]
           conn.close()
       except Exception as e:
           return f"Error connecting to MySQL: {e}", 500

       # Increment Redis counter
       try:
           r.incr('visit_count')
           redis_count = r.get('visit_count').decode('utf-8')
       except Exception as e:
           return f"Error connecting to Redis: {e}", 500

       return f"Total visits (MySQL): {count}, Total visits (Redis): {redis_count}"

   @app.route('/visit', methods=['POST'])
   def visit():
       # Connect to MySQL
       try:
           conn = pymysql.connect(host=db_host, user=db_user, password=db_password, database=db_name)
           cursor = conn.cursor()
           cursor.execute("INSERT INTO visits (visitor) VALUES (%s)", ('guest',))
           conn.commit()
           conn.close()
       except Exception as e:
           return f"Error connecting to MySQL: {e}", 500

       return "Visit recorded!"

   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=5000)
   ```

   Replace `'yourpassword'` with the password you set for the RDS instance.

5. **Create the MySQL table:**

   Connect to your MySQL RDS instance using a MySQL client.
   Use the MySQL CLI you installed in Prereqs. In a terminal (bash or PowerShell), run:

   ```bash
   mysql -h <RDS‑ENDPOINT> -P 3306 -u admin -p
   ```

   - Replace <RDS‑ENDPOINT> with the value Terraform output (e.g. webappdb-instance.cwbmawgukf0s.us-east-1.rds.amazonaws.com)
   - You’ll be prompted for the password you set (yourpassword).
   - Once connected, run SQL as usual (e.g. SHOW DATABASES;).

    and select your app database:

   ```sql
   USE webappdb
   ```
   Then create a new table by running the following SQL command:

   ```sql
   CREATE TABLE visits (
       id INT AUTO_INCREMENT PRIMARY KEY,
       visitor VARCHAR(255),
       visit_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );
   ```

---

## 🚀 Running the Application

1. **Set environment variables:**

   Bash (Linux/macOS):
   ```bash
   export DB_HOST='your-rds-endpoint'
   export DB_USER='admin'
   export DB_PASSWORD='yourpassword'
   export DB_NAME='webappdb'
   export REDIS_HOST='your-redis-hostname'
   export REDIS_PORT=6380
   export REDIS_PASSWORD='your-redis-access-key'
   ```

   PowerShell (Windows):
   ```powershell
   $Env:DB_HOST        = 'your-rds-endpoint'
   $Env:DB_USER        = 'admin'
   $Env:DB_PASSWORD    = 'yourpassword'
   $Env:DB_NAME        = 'webappdb'
   $Env:REDIS_HOST     = 'your-redis-hostname'
   $Env:REDIS_PORT     = '6380'
   $Env:REDIS_PASSWORD='your-redis-access-key'
   ```

   Replace the placeholder values with the actual values obtained from the Terraform outputs. You can also get the REDIS credentials from the Azure Portal by navigating to your resource then selecting the Authentication tab.

2. **Run the Flask application:**

   ```bash
   python app.py
   ```

   The application will start on `http://localhost:5000`.

---

## 🧪 Testing the Application

- **Record a visit:**

  ```bash
  curl -X POST http://localhost:5000/visit
  ```

- **Check visit counts:**

  Open `http://localhost:5000/` in your browser. You should see the total visits recorded in MySQL and Redis.

---

## 🧹 Cleanup

To avoid incurring charges:

- **AWS**: Navigate to the `aws_rds` directory and run:

  ```bash
  terraform destroy
  ```

- **Azure**: Navigate to the `azure_redis` directory and run:

  ```bash
  terraform destroy
  ```

---
