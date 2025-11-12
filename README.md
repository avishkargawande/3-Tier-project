
# 3-Tier Java Registration Form Deployment on AWS


This project showcases a Java-based Student Registration Web Application deployed using a    **Three-Tier Architecture**  on Amazon Web Services (AWS). The setup includes an **NGINX Proxy Server** in the public
 subnet, a **Tomcat Application Server** in the private subnet, and a **MariaDB Database** hosted on Amazon
 RDS. The Proxy Server not only handles HTTP requests but also acts as a Jump Server (Bastion Host) for
 secure SSH access to private-tier instances. 
The entire infrastructure is built inside a custom **VPC** using subnets, route tables, a NAT gateway, and security
 groups for secure communication between tiers. It demonstrates how traffic flows from users through a proxy
 to the application and database layers, following a real-world, scalable cloud deployment model.

* **Proxy Tier (Presentation Layer)**: Handles client requests, routes traffic, and ensures secure external access via NGINX.
* **Application Tier (Business Logic)**: Hosts the Java-based web application (Student Registration System)
 running on Apache Tomcat.
* **Database Tier (Data Layer)**: Stores user information in a relational database managed by Amazon RDS for reliability and automated backups. 



## Architecture Diagram:
![alt text](<Screenshot 2025-11-08 202838.png>)




### VPC Configuration
 
* VPC CIDR: 10.0.0.0/16
* Subnets:

        Public Subnet (Proxy): 10.0.0.0/20
        Private Subnet 1 (App): 10.0.16.0/20
        Private Subnet 2 (DB): 10.0.32.0/20

* NAT Gateway: In Public Subnet with Elastic IP
* Security Group Ports: 

        22 (SSH)
        80 (HTTP)
        8080 (Application)
        3306 (MySQL)       

###  AWS Setup Steps:
#### PART 1: Create Networking Resources

Step 1: Create a VPC

Create a VPC with range 10.0.0.0/16. This creates an isolated network to host all components securely.

![alt text](<Screenshot 2025-10-28 205045.png>)

 Step 2: Create Subnets
 Create three subnets:

*  Public Subnet → 10.0.0.0/20 (for Proxy Server)
*  Private Subnet 1 → 10.0.16.0/20 (for App Server)
* Private Subnet 2 → 10.0.32.0/20 (for DB Server)

![alt text](<Screenshot 2025-10-28 210340-1.png>)

Step 3: Create an Internet Gateway

Create an Internet Gateway and attach it to the VPC. This allows resources in the public subnet to connect to
 the internet. 
![alt text](<Screenshot 2025-10-28 205741.png>)

 Step 4: Configure Public Route Table

 Update Public Route Table to add an IGW route. 

![alt text](<Screenshot 2025-10-28 205950.png>)

Step 5: Create NAT Gateway

Create a NAT Gateway in the Public Subnet (auto-allocate Elastic IP). It allows private instances (App, DB) to access the internet 

![alt text](<Screenshot 2025-10-28 210518.png>)

 Step 6:

  Create Private Route table inside your VPC and add route of NAT gateway in it: Now private subnets can reach
 the internet through NAT. 


 Step 7:

 Associate Private Subnets to the private route table. This separates internal traffic from public access.
![alt text](<Screenshot 2025-10-28 210340-1.png>)

  Step 8:
   Create a Security Group with inbound rules:

   * 22 (SSH)
   * 80 (HTTP)
   * 8080 (Tomcat)
   * 3306 (MySQL/RDS)
  
![alt text](<Screenshot 2025-10-28 211021.png>)

 ###  PART 2: Launch EC2 Instances and Create RDS

![alt text](<Screenshot 2025-10-28 212526.png>)

  Also, create an RDS instance (MariaDB) in the same VPC with the same security group.

![alt text](<Screenshot 2025-10-28 213125.png>)

 ### PART 3: Proxy Server Setup (NGINX)

 Step 1 :Connect via SSH to the Proxy instance.

 Step 2 :Install and start NGINX: 

* sudo yum update -y
* sudo yum install nginx -y
* sudo systemctl start nginx
* sudo systemctl enable nginx
* sudo systemctl status nginx  

 Step 3 :Edit NGINX configuration:

 * cd /etc/nginx
 *  sudo vim nginx.conf

 Inside the server block, add: location / { proxy_pass http://:8080/student/; }

![alt text](<Screenshot 2025-10-28 214454-1.png>)

 *  sudo systemctl restart nginx
 NGINX will now forward external traffic to your Tomcat server.

###  PART 4: Application Server Setup (Tomcat)
 Step 1: From jump server(here Proxy) Connect to your App instance.

  Step 2: Install Java and Tomcat

  * update system
  *  install java
  *  install tomcat 

    sudo yum update -y 
    sudo yum install java -y 
    wget https://dlcdn.apache.org/tomcat/   tomcat-9/v9.0.98/bin/apache-tomcat
    9.0.98.tar.gz 
    sudo tar -xvzf apache-tomcat-9.0.98.tar.gz -C /opt 


Step 3:

 Check that tomcat is installed correctly:


 Step 4: Deploy your application WAR file:

  Deploy your web application inside webapps:

     cd /opt/apache-tomcat/webapps 
    wget <S3-Bucket-URL-to-App-Code> 


![alt text](<Screenshot 2025-10-28 221948.png>)
 Step 5: Restart Tomcat

     cd /opt/apache-tomcat/bin 
    ./catalina.sh stop 
    ./catalina.sh start


Step 6: Check Java

 Now open your browser to check java page:
*  http://Proxy-Public-IP 

![alt text](<Screenshot 2025-10-28 222419.png>)

 ### PART 5: Database Setup (MariaDB / RDS)
  Step 1:

  SSH into DB instance and take access of RDS:

     sudo yum install mariadb105-server -y 
     mysql -h <RDS-ENDPOINT> -u admin -p 

Step 2:

 Create database and table:

     CREATE DATABASE studentapp; 
    USE studentapp; 
    CREATE TABLE students ( 
    student_id INT NOT NULL AUTO_INCREMENT, 
    student_name VARCHAR(100) NOT NULL,
     student_addr VARCHAR(100) NOT NULL, 
     student_age VARCHAR(3) NOT NULL, 
    student_qual VARCHAR(20) NOT NULL, 
    student_percent VARCHAR(10) NOT NULL, 
    student_year_passed VARCHAR(10) NOT NULL, 
    PRIMARY KEY (student_id) 
    );   

 Exit MySQL:

         exit; 

### PART 6: Connect App Server to RDS
 Step 1:

  Install JDBC connector in App server:

     sudo -i 
    cd /opt/apache-tomcat/lib 
    wget <S3-Bucket-URL-to-JDBC-Connector>

![alt text](<Screenshot 2025-10-28 225919.png>)

 Step 2:

  Edit the context file:

     cd /opt/apache-tomcat/conf 
    sudo vim context.xml 


Step 3:

 Add this configuration inside context block:

         <Resource name="jdbc/TestDB"        auth="Container" 
        type="javax.sql.DataSource" 
          maxTotal="500" maxIdle="30" maxWaitMillis="1000" 
          username="admin" password="redhat123!" 
          driverClassName="com.mysql.jdbc.Driver" 
          url="jdbc:mysql://<RDS-ENDPOINT>:3306/studentapp?
 useUnicode=yes&characterEncoding=utf8"/> 

![alt text](<Screenshot 2025-10-30 153612.png>)

 Step 4:

  Restart Tomcat:

    cd /opt/apache-tomcat/bin 
    ./catalina.sh stop 
    ./catalina.sh start 

### PART 7: Access the Application and add entries

 Once all services are running visit: http://Proxy-Public-IP 

![alt text](<Screenshot 2025-10-30 153327.png>)

![alt text](<Screenshot 2025-10-30 153349.png>)


###  PART 8: Verify entries in the RDS
![alt text](<Screenshot 2025-10-30 154141.png>)

### Final Result

 Successfully deployed a 3-Tier Web Application on AWS with:

* Isolated VPC architecture
* Secure public and private subnets
* NAT Gateway for private internet access
* Reverse proxy with NGINX
* RDS integration for database storage
* End-to-end functional Student Registration System
   