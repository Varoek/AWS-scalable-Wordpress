# AWS-scalable-Wordpress
it is begginer friendly aws wordpress deployment full, from VPC to CloudWatch

1. Create a VPC with Public Subnet

    Go to VPC > Create VPC

        Name: wordpress-vpc

        IPv4 CIDR: 10.0.0.0/16

        DNS hostnames: enabled

    Create a subnet:

        Name: public-subnet

        Subnet CIDR: 10.0.1.0/24

        AZ: pick 1 (e.g., ap-southeast-1a)

    Create an Internet Gateway

        Attach to your VPC

    Create a Route Table

        Associate it with the public subnet

        Add route: 0.0.0.0/0 → Internet Gateway

2. Launch EC2 Instance (Ubuntu 22.04)

    Instance type: t2.micro (Free Tier)

    AMI: Ubuntu Server 22.04

    Network: Select your custom VPC + public subnet

    Public IP: Enable

    Storage: 10 GB gp2 or more

    Security group:

        Allow: 22 (SSH), 80 (HTTP), 443 (HTTPS)

🔐 Use your existing .pem key pair.
3. Install NGINX + PHP + WordPress

SSH into the instance:

sudo apt update && sudo apt upgrade -y
sudo apt install nginx php php-fpm php-mysql unzip curl mysql-client -y

Download and configure WordPress:

cd /var/www/
sudo curl -O https://wordpress.org/latest.zip
sudo unzip latest.zip
sudo chown -R www-data:www-data wordpress
sudo chmod -R 755 wordpress

Create NGINX config:

sudo nano /etc/nginx/sites-available/wordpress

Paste:

server {
    listen 80;
    server_name _;

    root /var/www/wordpress;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}

Then:

sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

4. Create RDS MySQL (Free Tier)

    Engine: MySQL 8.0

    Instance class: db.t3.micro

    Storage: 20 GB (Free Tier)

    VPC: same as EC2

    Public access: YES

    Subnet group: use default or create one with 2 AZs

    Security group: allow MySQL (3306) from EC2’s SG

Save DB info:

    Endpoint

    Username

    Password

    DB name

From EC2, test connection:

mysql -h <rds-endpoint> -u admin -p

5. Configure WordPress to Use RDS

Edit /var/www/wordpress/wp-config.php:

define('DB_NAME', 'your_db_name');
define('DB_USER', 'admin');
define('DB_PASSWORD', 'your_password');
define('DB_HOST', 'your-rds-endpoint');

Then go to http://your-ec2-ip in browser to finish setup.
6. Create IAM Role for S3

    Go to IAM > Roles > Create Role

        Service: EC2

        Attach policy: AmazonS3FullAccess (you’ll restrict this later)

    Attach to EC2 instance

7. Create S3 Bucket for Media Offload

    Name: your-wp-media-bucket

    Block Public Access: keep blocked

    Enable versioning (optional)

You can later use plugin like:

    Media Offload Lite or

    WP Offload Media

8. Enable CloudWatch Agent

SSH into EC2:

sudo apt install amazon-cloudwatch-agent -y
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

Sample config (replace instance ID later):

{
  "metrics": {
    "namespace": "EC2/WordPress",
    "metrics_collected": {
      "cpu": { "measurement": ["cpu_usage_idle"], "metrics_collection_interval": 60 },
      "disk": { "measurement": ["used_percent"], "metrics_collection_interval": 60 }
    }
  }
}

Start it:

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
