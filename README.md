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
### Important note
This really ONLY works if you have a domain name, and that will cost you some money.

For our public domain name to use the reader nodes, go to route 53 and select it, and choose ```Create Recordset```

Choose Alias: Yes and then choose your ALB from the list, and click ```Create```.

## Register your EC2 instance as target for your ALB

If you didn't register your EC2 instance as a target when you created your ALB, go back to EC2 - Target Groups, select your target group and click on the ```Targets``` tab.

If you have no registered targets listed, click ```Edit```, select your instance, and click ```Add to Register``` and ```Save```.


## Create Read / Write Nodes

### Read Node
We want one EC2 instance to be our reader instance, all blog posts will be read on this instance. It will continually sync up with the S3 code bucket to keep it current, pulling files down from the bucket.
To do this we are going to use a cron job.
```bash
cd /etc
nano crontab
```
Add the following:
```
*/1 * * * * root aws s3 sync --delete s3://yourcodebucketname /var/www/html
```
CTRL-X, Y and enter to save and exit.


#### Create an image

Go to EC2 and select our instance, from actions choose Image > Create Image

Give it a name identifying it as the default read node for our WP site, select ``` no reboot``` and click ```Create Image```.

Now we have an AMI for our fleet of readers.

### Write Node

Now that we have created an AMI for our read node, edit the crontab for the write node.
```bash
cd /etc
nano crontab
```
We want this to push to the S3 bucket to ensure everything is current. Also push to the media bucket for our CloudFront distribution.
```
*/1 * * * * root aws s3 sync --delete /var/www/html s3://yourcodebucketname 
*/1 * * * * root aws s3 sync --delete /var/www/html/wp-content/uploads s3://yourmediabucketname
```

This will be the instance our bloggers will use to write blogs.

### Create our Readers from the image 

#### Create a Launch Configuration

Go to Auto Scaling group and click create. Then click the link ```Create a new launch configuration```

On the left select MyAMIs and select our read node image.

Accept t2.micro, click ```Next: ...```

Name it and select our S3 role for the IAM role, as it needs to be using our bucket.

Add the following bootstrap script:
```bash
#!/bin/bash
yum update -y
# update with the wp code
aws s3 sync --delete s3://yourcodebucketname /var/www/html

```
So instances launched in our autoscaling group will update and grab the latest copy of the wp from the code bucket. 

Proceed, accept default storage and then choose our dmz security group, and hit ```Review```, then click ```Create launch configuration```, select/create a keypair file.

In the ```Create Auto Scaling Group``` page, name it and select the groups size you desire. Accept our default VPC, and select **all** the subnets in your region.

In the ```Advanced Details``` settings select ```Receive traffic from one or more load balancers``` and select our reader group in ```Target Groups```

> I chose ELB for Health Check type, pick your preference and accept settings

Click ```Next: Configure scaling policies```, default is ```Keep this group at its initial size```, use it unless you need to adjust the capacity of the group with policies (based on cpu, etc)

Click ```Next: Configure Notifications```, add if you wish, add a name in the tags so you remember what this is (your word press reader node), and click ```Review```. 

Should have something like this.

    Group name YourReaderNodesName
    Group size 2
    Minimum Group Size 2
    Maximum Group Size 2
    Subnet(s) subnet-9c05abe6,subnet-84b12cec
    Load Balancers 
    Target Groups yourWordPressInstanceGroup
    Health Check Type ELB
    Health Check Grace Period 300
    Detailed Monitoring No
    Instance Protection None
    Service-Linked Role AWSServiceRoleForAutoScaling

Go now to our Target Groups, and remove our Writer node from it, as we don't really want it getting read requests.

Select your wp instance group, click the Targets tab and click ```Edit```. From the ```Registered Targets``` select your writer node and click ```Remove```.

Ideally these would be the ones connected to your DNS name, so public traffic going to view your blog will be directed to the reader node instances. Your bloggers would connect to your writer node to create new blogs posts and uploads. The respective cron jobs on the servers will either pull from (reader node) or push to (writer node) your S3 buckets.

# Updating WP Plugins / Installing new Themes

In your writer EC2 instance, ssh in an run this to allow your word press site to update plugins and install new themes:

```bash
sudo chown -R apache:apache /var/www/html
```
> Thanks to [Basil](https://stackoverflow.com/users/1103128/basil-abbas) who answered this [question](https://stackoverflow.com/questions/17768201/wordpress-on-ec2-requires-ftp-credentials-to-install-plugins)

















