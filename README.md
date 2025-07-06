# Automated-EBS-Snapshot-and-S3-Versioning-Lab
This repository contains a detailed walkthrough of the AWS hands‑on lab for:  Automating EBS snapshots via cron  Managing S3 buckets with versioning  Syncing, deleting, and restoring file versions in S3  All lab screenshots (32 images) are stored in images/ and numbered chronologically. Replace the placeholders below with the actual file names.
Table of Contents

Setup: EC2 Instances and IAM

Discover Instance & Volume

Stopping the Instance

Creating Initial Snapshot

Automating Snapshots with Cron

Cleaning Up Old Snapshots

Syncing Files to S3 Bucket

Deleting & Restoring with Versioning

Conclusion

Setup: EC2 Instances and IAM

Create two EC2 instances: Command Host and Processor.

Attach an IAM role with AmazonEC2FullAccess and AmazonS3FullAccess policies.

Ensure the Processor instance has the AWS CLI configured.

Discover Instance & Volume

Identify the Processor instance by tag, then retrieve its root volume ID:

aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=Processor" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text

aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=Processor" \
  --query "Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId" \
  --output text

Stopping the Instance

Stop the instance, then wait until fully stopped:

aws ec2 stop-instances --instance-ids i-XXXXXXXXXX
aws ec2 wait instance-stopped --instance-ids i-XXXXXXXXXX

Creating Initial Snapshot

Create a snapshot of the root volume:

aws ec2 create-snapshot \
  --volume-id vol-YYYYYYYYYY \
  --description "Initial snapshot of Processor"

Verify completion:

aws ec2 describe-snapshots \
  --filters Name=volume-id,Values=vol-YYYYYYYYYY \
  --query "Snapshots[?State=='completed'].[SnapshotId,StartTime]" \
  --output table

Automating Snapshots with Cron

Add a cron job to run every minute (for demo) that appends the snapshot command to /tmp/cronlog:

(crontab -l ; echo "* * * * * aws ec2 create-snapshot --volume-id vol-YYYYYYYYYY >> /tmp/cronlog 2>&1") | crontab -

Check /tmp/cronlog every minute to see new snapshots.

Cleaning Up Old Snapshots

Use a Python script (snapshotter_v2.py) to delete all but the two most recent snapshots:

python3 snapshotter_v2.py

List remaining snapshots:

aws ec2 describe-snapshots \
  --filters Name=volume-id,Values=vol-YYYYYYYYYY \
  --query "Snapshots[*].SnapshotId" --output text

Syncing Files to S3 Bucket

Lab directory contains files.zip. On the Processor instance:

wget <files.zip URL>
unzip files.zip
export BUCKET=my-ebs-backups-12345
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
aws s3 sync files/ s3://$BUCKET/files/

Verify upload:

aws s3 ls s3://$BUCKET/files/

Deleting & Restoring with Versioning

Delete a local file and sync with --delete:

rm files/file1.txt
aws s3 sync files/ s3://$BUCKET/files/ --delete
aws s3 ls s3://$BUCKET/files/

List object versions and identify the version you want to restore:

aws s3api list-object-versions --bucket $BUCKET --prefix files/file1.txt

Get object for the old version:

aws s3api get-object --bucket $BUCKET --key files/file1.txt --version-id <VERSION_ID> files/file1.txt

Re-sync to push restored version as newest:

aws s3 sync files/ s3://$BUCKET/files/
aws s3 ls s3://$BUCKET/files/

Conclusion

You’ve successfully:

Automated EBS snapshots with AWS CLI and cron

Managed snapshot lifecycle and cleanup

Configured S3 bucket versioning

Synced, deleted, listed, and restored object versions in S3

Feel free to explore automating these tasks further with Lambda or Step Functions!

