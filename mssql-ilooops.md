To achieve a successful DB migration from an existing setup to a new MSSQL database, specifically with CodeIgniter and Docker, follow the structured steps below. These instructions include setting up a separate endpoint with a new branch dedicated to the MSSQL integration.

### 1. **Setup a New Branch for MSSQL Endpoint**

- **Create a new branch** in your version control system (e.g., Git) to isolate changes related to MSSQL integration:
  ```bash
  git checkout -b feature/mssql-endpoint
  ```

- Ensure this branch is dedicated to any changes needed for the MSSQL database migration. This will allow you to keep the original setup intact while testing and deploying the new configuration.

### 2. **Configure MSSQL Database Instance**

- **Database Setup**: Ensure you have access to an MSSQL server instance. This can be a local setup, a containerized instance, or a cloud-hosted database. Here's a quick setup using Docker for MSSQL:
  
  ```bash
  docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=YourStrongPassword!' \
    -p 1433:1433 --name sqlserver \
    -d mcr.microsoft.com/mssql/server:2019-latest
  ```
  
- **Create Database & User**: Once the MSSQL instance is running, connect using a SQL client (like `sqlcmd`, Azure Data Studio, or SQL Server Management Studio) and create the necessary database and user.
  
  ```sql
  CREATE DATABASE YourDatabaseName;
  CREATE LOGIN YourUserName WITH PASSWORD = 'YourStrongPassword!';
  CREATE USER YourUserName FOR LOGIN YourUserName;
  ALTER ROLE db_owner ADD MEMBER YourUserName;
  ```

### 3. **Install PHP MSSQL Driver**

- Update the **Dockerfile** to include the PHP extension for MSSQL:
  
  ```dockerfile
  # Install Microsoft ODBC Driver and PHP extensions for MSSQL
  RUN apt-get update && \
      apt-get install -y unixodbc-dev gnupg2 && \
      curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - && \
      curl https://packages.microsoft.com/config/ubuntu/20.04/prod.list > /etc/apt/sources.list.d/mssql-release.list && \
      apt-get update && ACCEPT_EULA=Y apt-get install -y msodbcsql17 && \
      pecl install sqlsrv pdo_sqlsrv && \
      echo "extension=sqlsrv.so" > /etc/php/5.6/apache2/conf.d/30-sqlsrv.ini && \
      echo "extension=pdo_sqlsrv.so" > /etc/php/5.6/apache2/conf.d/30-pdo_sqlsrv.ini && \
      apt-get clean
  ```

- Rebuild your Docker image to incorporate these changes:
  
  ```bash
  docker build -t codeigniter-mssql .
  ```

### 4. **Configure CodeIgniter for MSSQL Connection**

1. **Database Configuration**: Update the database configuration to connect to the MSSQL server. In the CodeIgniter project, modify the `application/config/database.php` file:
   
   ```php
   $db['default'] = array(
       'dsn'	=> '',
       'hostname' => 'your_sql_server_hostname_or_ip',
       'username' => 'YourUserName',
       'password' => 'YourStrongPassword!',
       'database' => 'YourDatabaseName',
       'dbdriver' => 'sqlsrv',
       'dbprefix' => '',
       'pconnect' => FALSE,
       'db_debug' => (ENVIRONMENT !== 'production'),
       'cache_on' => FALSE,
       'cachedir' => '',
       'char_set' => 'utf8',
       'dbcollat' => 'utf8_general_ci',
       'swap_pre' => '',
       'encrypt' => FALSE,
       'compress' => FALSE,
       'stricton' => FALSE,
       'failover' => array(),
       'save_queries' => TRUE
   );
   ```

2. **Environment-specific Configuration**: If you have environment-based configuration files (like `application/config/development/database.php`), make sure the changes reflect there as well.

### 5. **Database Migration with CodeIgniter**

1. **Enable Migrations**: In `application/config/migration.php`, set the migration preferences:
   
   ```php
   $config['migration_enabled'] = TRUE;
   $config['migration_type'] = 'sequential'; // or 'timestamp' depending on your preference
   $config['migration_table'] = 'migrations';
   ```

2. **Create Migration File**:
   
   - Create a new migration file using CodeIgniter's migration system (e.g., `001_add_new_table.php`):
     
     ```bash
     php index.php migrate create AddNewTable
     ```

   - In the new file (`application/migrations/001_add_new_table.php`), define the schema:
     
     ```php
     defined('BASEPATH') OR exit('No direct script access allowed');

     class Migration_AddNewTable extends CI_Migration {

         public function up() {
             $this->dbforge->add_field(array(
                 'id' => array(
                     'type' => 'INT',
                     'constraint' => 5,
                     'unsigned' => TRUE,
                     'auto_increment' => TRUE
                 ),
                 'title' => array(
                     'type' => 'VARCHAR',
                     'constraint' => '100',
                 ),
                 'description' => array(
                     'type' => 'TEXT',
                     'null' => TRUE,
                 ),
             ));
             $this->dbforge->add_key('id', TRUE);
             $this->dbforge->create_table('new_table');
         }

         public function down() {
             $this->dbforge->drop_table('new_table');
         }
     }
     ```

3. **Run Migration**:
   
   ```bash
   php index.php migrate
   ```

### 6. **Create a New Endpoint for MSSQL-Specific Functionality**

- Create a new controller in `application/controllers` for the MSSQL-specific endpoint. For example, `MssqlEndpoint.php`:
  
  ```php
  defined('BASEPATH') OR exit('No direct script access allowed');

  class MssqlEndpoint extends CI_Controller {

      public function __construct() {
          parent::__construct();
          $this->load->database(); // Load the database configuration set for MSSQL
      }

      public function index() {
          $query = $this->db->query("SELECT * FROM your_table");
          $data = $query->result();
          echo json_encode($data);
      }
  }
  ```

- Access this endpoint via the URL like:
  ```
  http://your-domain.com/index.php/mssqlendpoint
  ```

### 7. **Testing & Deployment**

- **Testing**: Ensure that all changes are tested locally. Use tools like Postman to verify that the new endpoint communicates correctly with the MSSQL database.

- **Deployment**:
  - Merge the `feature/mssql-endpoint` branch into the main branch once testing is successful.
  - Deploy your Docker container to the desired environment (e.g., AWS, Azure, or local server).

By following these steps, you’ll establish a fully functioning setup with a dedicated MSSQL database and endpoint while maintaining the original database's integrity.


---

To update your Dockerfile to include the PHP extensions necessary for MSSQL, I will integrate the commands to install the **Microsoft ODBC Driver**, **PHP `sqlsrv`**, and **PDO `sqlsrv`** extensions. These are crucial for enabling MSSQL support in a PHP environment. Here’s the revised Dockerfile:

### **Updated Dockerfile**
```dockerfile
FROM ubuntu:20.04

# Installing Apache and PHP5.6
RUN apt-get update && apt-get install -y software-properties-common && \
    add-apt-repository -y ppa:ondrej/php && \
    apt-get update -y && apt-get install -y apache2 vim wget build-essential checkinstall zlib1g-dev \
    php5.6 \
    libapache2-mod-php5.6 \
    php5.6-common \
    php5.6-mysql \
    php5.6-gmp \
    php5.6-ldap \
    php5.6-json \
    php5.6-curl \
    php5.6-intl \
    php5.6-mbstring \
    php5.6-xmlrpc \
    php5.6-gd \
    php5.6-bcmath \
    php5.6-xml \
    php5.6-cli \
    php5.6-zip \
    unixodbc-dev gnupg2 curl

# Installing Microsoft ODBC Driver and PHP extensions for MSSQL
RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - && \
    curl https://packages.microsoft.com/config/ubuntu/20.04/prod.list > /etc/apt/sources.list.d/mssql-release.list && \
    apt-get update && ACCEPT_EULA=Y apt-get install -y msodbcsql17 mssql-tools && \
    echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc && \
    source ~/.bashrc && \
    apt-get install -y php-pear && \
    pecl install sqlsrv-5.6.1 pdo_sqlsrv-5.6.1 && \
    echo "extension=sqlsrv.so" > /etc/php/5.6/apache2/conf.d/30-sqlsrv.ini && \
    echo "extension=pdo_sqlsrv.so" > /etc/php/5.6/apache2/conf.d/30-pdo_sqlsrv.ini

# OpenSSL Downgrade
WORKDIR /usr/local/src/
RUN wget https://www.openssl.org/source/openssl-1.1.1i.tar.gz && \
    tar -xf openssl-1.1.1i.tar.gz && \
    cd /usr/local/src/openssl-1.1.1i && \
    ./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl shared zlib && \
    make && \
    make install && \
    cd /etc/ld.so.conf.d/ && \
    touch openssl-1.1.1i.conf && \
    echo "/usr/local/ssl/lib" > openssl-1.1.1i.conf && \
    ldconfig -v && \
    mv /usr/bin/c_rehash /usr/bin/c_rehash.BACKUP && mv /usr/bin/openssl /usr/bin/openssl.BACKUP && \
    echo 'PATH="$PATH:/usr/local/ssl/bin"' | tee -a /etc/environment

# SSL terminate in php.ini
RUN wget https://curl.haxx.se/ca/cacert.pem --no-check-certificate && \
    echo "curl.cainfo="/usr/local/src/cacert.pem"" > /etc/php/5.6/apache2/php.ini && \
    echo "openssl.cafile="/usr/local/src/cacert.pem"" > /etc/php/5.6/apache2/php.ini

# Copy codebase
WORKDIR /var/www/html
COPY ./ ./ 
RUN rm index.html

# Copy conf file
COPY ./docker/000-default.conf /etc/apache2/sites-available/000-default.conf
COPY ./docker/php.ini /etc/php/5.6/apache2/php.ini

# Enable apache modules
RUN a2enmod rewrite

RUN chmod +x ./docker/bootstrap.sh

RUN chmod +x ./docker/entrypoint.sh

EXPOSE 80

ENTRYPOINT [ "./docker/entrypoint.sh" ]
```

### **Key Changes Explained**

1. **Microsoft ODBC Driver Installation**:
   - Added the repository and installed `msodbcsql17` and `mssql-tools` for MSSQL connectivity.

2. **PHP MSSQL Extensions**:
   - Installed `sqlsrv` and `pdo_sqlsrv` extensions using `pecl`.
   - Created configuration files (`30-sqlsrv.ini` and `30-pdo_sqlsrv.ini`) to enable these extensions in PHP.

3. **Dependency Updates**:
   - Installed `unixodbc-dev` and `gnupg2` as prerequisites for the ODBC and PHP extension installations.

### **Testing**

- After updating the Dockerfile, rebuild your image:
  
  ```bash
  docker build -t codeigniter-mssql .
  ```

- Run the container and verify that the `sqlsrv` and `pdo_sqlsrv` extensions are loaded by checking the PHP info page or using the command:

  ```bash
  docker exec -it <container_id> php -m | grep sqlsrv
  ```

By following this updated Dockerfile, you’ll have a PHP environment that fully supports MSSQL, allowing you to connect and operate with a Microsoft SQL Server database.
