Building an Automated Business Process using Managed Services on a Public Cloud

In the connected world, it is imperative that the organizations be interlinked with the customers and vendors. 
This process has been very sluggish, manual, batch-based and prone to failures. 
Such Integration design has lead to impaired decision-making and delays in the detection of fraudulent actions. 
This project created an automated, event-based real-time process using managed cloud services that do not have these limitations.

Heew is thw architecture👇
<img src="https://github.com/hisujata/Building-an-Automated-Business-Process-using-Managed-Services-on-a-Public-Cloud/blob/master/Architecture.png">

Services that I have used AWSS3, AWSSNS, AWSRDS, Python, Boto3

Step 1: SNS and S3 topic creation

#a. Creation of Source and target buckets

1) Navigate to S3 using the Services button at the top of the screen
2) Select "Create Bucket"
3) Enter a source bucket name and use the default options for the rest of the fields
4) Click on "Create Bucket'
5) Repeat the above steps to create a target bucket

<img src="https://github.com/hisujata/Building-an-Automated-Business-Process-using-Managed-Services-on-a-Public-Cloud/blob/master/screenshot1.png">

#b. Creation of SNS subscription

1) Navigate to SNS -> Topics
2) Click on "Create Topic"
3) Enter the following fields
Name : S3toEC2Topic
The other options can be ignored for now
4) Click on Create Topic

<img src="https://github.com/hisujata/Building-an-Automated-Business-Process-using-Managed-Services-on-a-Public-Cloud/blob/master/screenshot3.png">

#c. Modification of SNS Access Policy

1) Navigate to SNS -> Topics and select the topic created in the previous step
2) Note down the ARN shown in the topic details
2) Click on Edit and select "Access Policy".
3) Replace the text in the JSON editor with the following

{
"Version": "2012-10-17",
"Id": "example-ID",
"Statement": [
{
"Sid": "example-statement-ID",
"Effect": "Allow",
"Principal": {
"AWS":"*"
},
"Action": [
"SNS:Publish"
],
"Resource": "SNS-topic-ARN",
"Condition": {
"ArnLike": { "aws:SourceArn": "arn:aws:s3:*:*:bucket-name" },
"StringEquals": { "aws:SourceAccount": "bucket-owner-account-id" }
}
}
]
}

4) Replace the bold text with the SNS topic ARN, source bucket name and your AWS account ID respectively.
5) Click on Save Changes


#d. Configuring SNS notifications for S3

1) Navigate to S3 and select the source bucket created in Step 1 (a)
2) Select Properties and scroll down to Event Notifications and select it
3) Select "Create Event Notification"
4) Fillup the details as follows
Name : S3PutEvent
Select PUT from the list of radio buttons
Destination : Select SNS Topic
SNS : Select S3ToEC2Topic
5) Save Changes

<img src="https://github.com/hisujata/Building-an-Automated-Business-Process-using-Managed-Services-on-a-Public-Cloud/blob/master/screenshot5.png">

Step 2: Run the custom program in the EC2 instance

#a. Creation of the EC2 instance and RDS instance

1) Navigate to EC2 -> Instances
2) Create an EC2 instance with the following parameters
AMI : Amazon Linux 2
VPC : Default
Security group : Ports 22 and 8080 should be opened


3) Navigate to RDS
4) Create an RDS instance with the following parameters: 

Engine type : MySql
Template : Dev/Test
Set the username and password as required
DB Instance class : Burstable
Instance type : t3.micro 
Public Access : Yes
VPC Security group : Create New ()

Under Additional Configuration, add an initial database name. Take note of this name as it will be required later. 

Uncheck “Enable Enhanced Monitoring”  

Ensure that the security group created by the RDS deployment has port 3306 open for all incoming connections from all sources. 

<img src="https://github.com/hisujata/Building-an-Automated-Business-Process-using-Managed-Services-on-a-Public-Cloud/blob/master/screenshot6.png">

#b. Assignment of IAM role for EC2 instance

1) Navigate back to EC2- > Instances
2) Select the EC2 instance created in the previous step and select Actions-> Security -> Modify IAM role
3) Select the role LabInstanceProfile from the dropdown and click on Save

<img src="https://github.com/hisujata/Building-an-Automated-Business-Process-using-Managed-Services-on-a-Public-Cloud/blob/master/screenshot7.png">

#c. Configuration and Uploading of custom program

1) Download the file docproc-new.zip on your machine 
2) Unzip the downloaded file
3) Enter the unzipped folder and open the file views.py in the API folder using a text editor
4) In line number 19-24, modify the target bucket name to the one created in Step 2 (a)  and modify the hostname, username, password and database variables to the values set while creating the RDS database and save the file
5) Copy the folder docproc-new to the home folder of the EC2 instance created in Step 3(a) using scp. Use the command given below
scp -i <pem> -r ./docproc-new ec2-user@<ip>:/home/ec2-user


<img src="https://github.com/hisujata/Building-an-Automated-Business-Process-using-Managed-Services-on-a-Public-Cloud/blob/master/screenshot8.png">

Step 3: Creation and Verification of SNS subscription and Generation of CSV file

#a. Starting the EC2 custom program

1) Log into the EC2 instance using SSH
2) Run the followng commands after successful SSH to start the server
sudo cp -r docproc-new /opt
sudo chown ec2-user:ec2-user -R /opt
cd /opt/docproc-new
sudo yum update
sudo yum install python-pip -y
python -m pip install --upgrade pip setuptools
sudo pip install virtualenv
virtualenv ~/.virtualenvs/djangodev
source ~/.virtualenvs/djangodev/bin/activate
pip install django
pip install boto3
pip install mysql-connector-python-rf
python manage.py runserver 0:8080

Keep this terminal window open throughout the rest of the exercise

<img src="">

#b. Creation of SNS subscription

1) Navigate to SNS in the AWS Console and select the topic S3ToEC2Topic
2) Click on Create Subscription
3) Enter the following details
Protocol : HTTP
Endpoint : http://<host>:8080/sns where <host> in the public IP of the EC2 instance
Click on Create Subscription
4) In the EC2 terminal window, look for the field "SubscribeURL" and copy the entire link given
Note: If a message is seen "ValueError: No JSON object could be decoded", it can be safely ignored
5) Paste that link into a browser window to verify the SNS subscription (Ignore any messages received in the web browser)

<img src="">

#c. Generation of CSV file

1) Download the file docproc-invoice.txt provided with this workbook
2) Navigate to S3 in the AWS Console
3) Upload the sample invoice file to the source S3 bucket using the default options
4) Verify that a CSV file is generated in the target S3 bucket. This may take a few minutes
5) (Optional) Login to the RDS instance using your preferred MySQL client and check the table created inside the specified database. 
1) Generated CSV file in the target S3 bucket

<img src="https://github.com/hisujata/Building-an-Automated-Business-Process-using-Managed-Services-on-a-Public-Cloud/blob/master/screenshot9.png">

Cloud is always pay per use model and all resources/services that we consume are chargeable. 
After completing the project, make sure to delete each resource created in reverse chronological order if you do not wish to use the resources.






