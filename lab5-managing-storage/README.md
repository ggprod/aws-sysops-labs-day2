# Lab 5: Managing Storage
In this lab you will create an instance and then take snapshots of its EBS volume and upload log files to an S3 bucket

## Task 1: Create an S3 bucket, a role, and an instance
In this task you will create an S3 bucket and an instance

1. In **Services->S3** click **Create bucket** for the **Name** use firstname-lastname-lab5, leave the **Region** and other settings as is and click **Next** several times and then **Create bucket**

2. In **Services->IAM** click **Roles**, then **Create Role**, then under **Select type of trusted entity** click **AWS service** and click **EC2**, then click **Next: Permissions** then **Next: Tags** (you didn't add a managed policy but will add your own custom one), then **Next: Review**, for **Role name** enter firstname-lastname-role, then click **Create role**

You will now add a custom (in-line) policy to grant the role access to the S3 bucket you created

3. Click on the role you just created, click **Add inline policy**, then click on the **JSON** tab, delete the lines that are there and paste in the below JSON (replace YOUR-BUCKET-NAME with the name of your bucket, there are two places to do that)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets",
                "s3:HeadBucket",
                "s3:ListBucket"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::YOUR-BUCKET-NAME/*",
                "arn:aws:s3:::YOUR-BUCKET-NAME"
            ]
        }
    ]
}
```

This policy grants full access to that bucket and any contents stored in the bucket

4. Click **Review policy** and provide the same name as the bucket name (firstname-lastname-lab5), then click **Create policy**

5. In **Services->EC2**, create an instance via **Launch instance** and select Amazon Linux 2 and t2.micro, set the **Iam role** to be the role you created, leave everything else as is except give the instance a tag that has key equal to Name and value set to firstname-lastname-l5-proc, click **Launch** then select **Create a new key pair** name the key pair firstname-lastname-l5-proc and download it

6.  Create another instance with the same values except without setting the **Iam role** and putting a tag with key set to Name and value set to firstname-lastname-l5-host, click **Launch** then select **Create a new key pair** name the key pair firstname-lastname-l5-host and download it then click **Launch Instance**

## Task 2: Taking snapshots of your instance EBS volume
In this task you will take a snapshot of the root EBS volumen of your instance

1.  Connect to the firstname-lastname-l5-host instance via SSH using the key pair (change to the folder that has the key, then `chmod 400 ./firstname-lastname-l5-host.pem`  then `ssh -i ./firstname-lastname-l5-host.pem ec2-user@public-ip-address`).  Remember to replace firstname-lastname with your firstname and lastname and public-ip-address with the public ip address of your instance

2. First run `aws configure` to set up the cli on the instance (you'll need to create a new access key first, delete an old one if you already have 2). Remember to set the default region to the same on you created your instances in. To get info about your firstname-lastname-l5-proc instance run the following command: `aws ec2 describe-instances --filter 'Name=tag:Name,Values=firsntame-lastname-l5-proc'` (remember to replace firstname-lastname)

3. To get only the data you want instead of the giant JSON response run the following command instead: `aws ec2 describe-instances --filter 'Name=tag:Name,Values=firstname-lastname-l5-proc' --query 'Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.{VolumeId:VolumeId}'`

This gives you just the volume id (copy it somewhere to use for later), which is what you want to be able to take a snapshot of that volume.  But to make sure you capture a consistent state of a boot disk with a snapshot you should stop the instance first (if it was a non-boot disk an unmount would be sufficient)

4. Stop the firstname-lastname-l5-proc instance via the following command: `aws ec2 stop-instances --instance-ids INSTANCE-ID` (replace INSTANCE-ID with the instance id of that instance which you can see on the instances page)

5. Execute a command to wait until the instance completes stopping: `aws ec2 wait instance-stopped --instance-id INSTANCE-ID` (replace INSTANCE-ID with same instance id as previous step).  When that command returns to the prompt it means the instance has stopped

6. Take a snapshot of the root EBS volume of the instance via the following command: `aws ec2 create-snapshot --volume-id VOLUME-ID` (replace VOLUME-ID with the volume id you copied previously).  Copy the SnapshotId field (you will use this later)

7. Execute the following command to wait until the snapshot completes: `ws ec2 wait snapshot-completed --snapshot-id SNAPSHOT-ID` (replace SNAPSHOT-ID with the value you copied).  When this command completes and returns to a prompt the snapshot has completed

8. Restart your firstname-lastname-l5-proc instance with this command: `aws ec2 start-instances --instance-ids INSTANCE-ID` (replace INSTANCE-ID with the instance id of that instance)

9. Run the following command to wait until the relaunch completes: `ws ec2 wait instance-running --instance-id INSTANCE-ID` (replace INSTANCE-ID as above)

Normally snapshots are something you take on a schedule and to do that in linux you can use the standard cron tool which we will now setup

10. To create a cron job that takes a snapshot every minute execute this command: `echo "* * * * *  aws ec2 create-snapshot --volume-id VOLUME-ID 2>&1 >> /tmp/cronlog" > cronjob` (replace VOLUME-ID with the volume id you copied earlier).  To start running the cron job run the following command: `crontab cronjob`

11. To verify the snapshots are being created execute the following command: `aws ec2 describe-snapshots --filters "Name=volume-id,Values=<volume-id>"` (replace <volume-id> with the volume id you copied earlier).  Wait a few minutes and execute again to see the growing number of snapshots.

Normally, you would also be running a script the periodically deleted all but the last few snapshots to free up storage space

12. Stop the cronjob via the command: `crontab -r`

## Task 3: Synchronize files on EBS volume with cloud storage
In this task we'll set up a synchronization of files between the EBS volume on the firstname-lastname-l5-proc instance and cloud storage

1. Connect via SSH to your firstname-lastname-l5-proc instance

2. Download a bunch of files into the instance via the command: `wget https://us-west-2-tcprod.s3.amazonaws.com/courses/ILT-TF-100-SYSOPS/v3.3.6/lab-5-storage-linux/scripts/files.zip`, then unzip the files via: `unzip files.zip`

Before setting up a file sync with your bucket you'll need to enable versioning on your bucket

3. Set up the AWS CLI on firstname-lastname-l5-proc via `aws configure` (You'll need another new Access key, delete one of the old ones to if you already have 2).  To enable versioning on your bucket use this command: `aws s3api put-bucket-versioning --bucket firstname-lastname-lab5 --versioning-configuration Status=Enabled` (remember to replace firstname-lastname-lab5 with your bucket name).

4. Execute the following command to sync: `aws s3 sync files s3://firstname-lastname-lab5/files/` (replace firstname-lastname-lab5 with your bucket name), verify it worked with the command: `aws s3 ls s3://firstname-lastname-lab5/files/`, then delete a local file via: `rm files/file1.txt`

5. To synchronize that deletion with the S3 bucket run the following command: `aws s3 sync files s3://firstname-lastname-lab5/files/ --delete` (replace firstname-lastname-lab5 with your bucket name), then verify it worked with the command: `aws s3 ls s3://firstname-lastname-lab5/files/`

6. To see the old verion of the file you deleted, execute the command: `aws s3api list-object-versions --bucket firstname-lastname-lab5 --prefix files/file1.txt`

There is no single command to directly restore a deleted file back to the bucket however you can do it as a multi-step process

7. To download the old pre-deleted verion of the file execute the following command: `aws s3api get-object --bucket firstname-lastname-lab5 --key files/file1.txt --version-id VERSION-ID files/file1.txt` (replace firstname-lastname as usual and VERSION-ID with the VersionId you saw in the previous command), verify the file was restored via `ls files`

8. To restore the file back to the S3 bucket run the command: `aws s3 sync files s3://firstname-lastname-lab5/files/` and verify it was restored to the S3 bucket with: `aws s3 ls s3://firstname-lastname-lab5/files/`

9. Exit the SSH session and delete all the resources you created (the instances, the S3 bucket and the role).