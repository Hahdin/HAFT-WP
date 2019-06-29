# Create a Highly Available, Fault Tolerant Wordpress Site on AWS
> assume valid aws account

## Setup our Buckets
Create 2 buckets, one for the wordpress code and another for the media.
- accept defaults

## CloudFront Distribution

### Create a web distribution for your media bucket
```Origin Domain Name``` set to your media bucket, leave the rest as defaults.

## Security Groups

Go to your VPC and select Security Groups

Create your default dmz sg with the following rules:

> HTTP on port 80 allow 0.0.0.0/0, ::/0

> SSL on port 22 allow 0.0.0.0/0

Create your rds security group with the following rules:

> MYSQL/Aurora on port 3306 for your IP

> MYSQL/Aurora on port 3306 for your dmz security group (sg- etc..)

## RDS

Go to RDS and create a new MySQL database

### Dev Test
For the dev test the following was used:

> Template : Dev/Test

> DB Instance Size: Burstable Classes (this included the t2.micro)

> Storage : General Purpose SSD

> Availability & durability : Multi-AZ Deployment    *this costs $

> VPC Security Options : select ```Choose existing VPC security groups``` and select our RDS security group.

> Database Options: add name
Fill in required fields (ie. username, passwords) and click Create Database.

## IAM Role

Create a role for your EC2 instance to talk to S3 if you don't have one.

From IAM, click ```Create Role```, select ```EC2```, click Next. Name it and select ```AmazonS3FullAccess```, proceed and save it.

## Provision our EC2 Instance

Go to EC2 and click ```Launch Instance```, select ```Amazon Linux 2 AMI (HVM), SSD Volume Type```, click ```Next: Configure Instance Details```.

Change IAM role to our new S3 role and add the following bootstrap under Advanced Details: User data:
```bash
#!/bin/bash
yum update -y
yum install httpd php php-mysql -y
cd /var/www/html
echo "healthy" > healthy.html
wget https://wordpress.org/wordpress-5.1.1.tar.gz
tar -xzf wordpress-5.1.1.tar.gz
cp -r wordpress/* /var/www/html/
rm -rf wordpress
rm -rf wordpress-5.1.1.tar.gz
chmod -R 755 wp-content
chown -R apache:apache wp-content
wget https://s3.amazonaws.com/bucketforwordpresslab-donotdelete/htaccess.txt
mv htaccess.txt .htaccess
chkconfig httpd on
service httpd start
```

Proceed with launch, choose your dmz security group and either create or use an existing keypair file.


### Note

The lab bucket provides the following htaccess.txt file:
```
Options +FollowSymlinks
RewriteEngine on
rewriterule ^wp-content/uploads/(.*)$ http://yourdistributionid.cloudfront.net/$1 [r=301,nc]

# BEGIN WordPress

# END WordPress
```

This file essentially allows wordpress to rewrite your urls to use your cloudfront distribution. 

Edit it to add your cloudfront distribution:
```bash
nano .htaccess
```
Change, CTRL-X, Y enter



SSH into your server and confirm apache is running

```bash
cd /var/www/html
ls
cat .htaccess
service httpd start
service httpd status
```

Now open your browser to your public IP address for your EC2 instance, and you should see the ```Welcome to WordPress``` page.

## Install WordPress

Click ```Let's Go!``` and fill in the information and click ```Submit```.

if you see the window with 

```
Sorry, but I can’t write the wp-config.php file.
```

copy the contents of the window to your clipboard and go to your terminal and create the file.

```bash
nano wp-config.php
```
Paste the contents, CTRL-X, Y enter

Go back to your wordpress page and click ```Run installation``` and you should now hopefully have a working word press site hosted in the AWS cloud.

## Increased Resiliency / CloudFront CDN

### Copy uploads to our S3 media bucket
from your terminal:

Make sure you can access S3
```bash
aws s3 ls
```
>> If you cannot, go make sure there is an S3 full access IAM role attached to your EC2

copy the contents of the uploads folder in your wordpress installation to s3
```bash
aws s3 cp --recursive /var/www/html/wp-content/uploads s3://mediabucketname
```
Now make an copy of the entire installation to save in our code bucket
```bash
aws s3 cp --recursive /var/www/html s3://codebucketname
```

If you change a file (like editing your .htaccess) you can update the S3:
```bash
aws s3 sync /var/www/html s3://codebucketname
```
If you create a post with new media, you can update the S3:
```bash
aws s3 sync /var/www/html s3://mediabucketname
```

### Let Apache know you are allowing url rewrites

Now we need to edit our httpd.conf file.
```bash
cd /etc/httpd/conf
# make a backup copy first
cp httpd.conf httpd-copy.conf
# open w/ nano
nano httpd.conf

```

Find this part of the file

    # AllowOverride controls what directives may be placed in .htaccess files.
    # It can be "All", "None", or any combination of the keywords:
    #   Options FileInfo AuthConfig Limit
    #
    AllowOverride None

Change 'None' to 'All' and this will allow url rewrites so we can use our cloudfront distribution.

CTRL X, Y enter to save and exit.

Restart the service.
```bash
service httpd restart
```

## Edit Media Bucket Policy to make images public

> Make sure you make your media bucket public first.
> - Edit Public Access, make sure its all unchecked, and cofirm

Go to your media bucket and add the following policy

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::yourmediabucketname/*"
            ]
        }
    ]
}
```

## Put our EC2 instance behind an application load balancer (ALB)

Go to EC2 and select Load Balancers, choose Application load balancer and click Create.

Give it a name and select all availablity zones in your region, click Next...

Continue past security warning (add https if you desire) and click ```Next: Configure Security Groups```.

Select your existing dmz security group, click ```Next: Configure Routing```

Create a new target group for your word press instance.

Adjust Advanced health check Settings:
- healthy thresh: 3
- unhealthy thresh: 2
- timeout: 5
- interval: 6

Click ```Next: Register Targets``` and select your EC2 instance, Review and Create.


## Route 53
If you have created a domain name, go to route 53 and select it, and choose ```Create Recordset```

Choose Alias: Yes and then choose your ALB from the list, and click ```Create```.

## Register your EC2 instance as target for your ALB

If you didn't register your EC2 instance as a target when you created your ALB, go back to EC2 - Target Groups, select your target group and click on the ```Targets``` tab.

If you have no registered targets listed, click ```Edit```, select your instance, and click ```Add to Register``` and ```Save```.



