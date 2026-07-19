# AWS-CloudDeployment

Activities done:
1.S3 Bucket and its actions:
2. Automated EBS SnapShot creation and Cleanup:
5. Revert Ec2 from snapshot
6. S3-SNS-Public Access

1.S3 Bucket and its actions:
Buckets created for testing purpose:
 <img width="975" height="216" alt="image" src="https://github.com/user-attachments/assets/6ec4bab9-2c1f-452e-a0c9-fa72c18e0a56" />


Trying to upload file in one of the bucket:  
<img width="975" height="488" alt="image" src="https://github.com/user-attachments/assets/8230923c-b0e0-444a-81e2-2655a0859a72" />

Uploaded successfully
<img width="975" height="358" alt="image" src="https://github.com/user-attachments/assets/75fb5cbb-4275-45d8-9173-857d2bfc0e97" />
<img width="975" height="232" alt="image" src="https://github.com/user-attachments/assets/78ab7aa6-5143-406c-a6a5-11e6dd073973" />


 
 
Trying to upload file using command 
<img width="975" height="191" alt="image" src="https://github.com/user-attachments/assets/4e6ba33c-8246-4b88-bb91-7af42423f428" />

 
Copied file using commands on another s3 bucket, listed the bucket and listed file in one of the bucket
<img width="975" height="124" alt="image" src="https://github.com/user-attachments/assets/6be1acfd-46c8-468f-8bbe-e396af2adb3f" />

 

Added respective Permissions on Lambda function:
<img width="975" height="576" alt="image" src="https://github.com/user-attachments/assets/abac3d38-312f-41b6-8876-d695391eafbc" />

 
Now I am able to see buckets created in S3
<img width="975" height="576" alt="image" src="https://github.com/user-attachments/assets/0d269323-2104-4a4c-9b32-84227a68cda4" />

 
Log Events:
<img width="975" height="463" alt="image" src="https://github.com/user-attachments/assets/e9ca0fba-fbec-4cd7-a2b0-7a86cfc21245" />

 
Used to get list of files in the S3 bucket:  
<img width="975" height="473" alt="image" src="https://github.com/user-attachments/assets/f1c2c69e-ed72-4ee1-92bb-b58821a9d49c" />


import json
import boto3

# Initialize the S3 client
s3 = boto3.client('s3')

def lambda_handler(event, context):
   
    TARGET_BUCKET = "senthil-bucket-18-07-2026-new" 
    
    try:
        file_list = []
        print(f"Fetching files from bucket: {TARGET_BUCKET}")
        
        # Using a paginator to make sure we fetch all files securely
        paginator = s3.get_paginator('list_objects_v2')
        page_iterator = paginator.paginate(Bucket=TARGET_BUCKET)
        
        for page in page_iterator:
            # Extract files from the page contents
            contents = page.get('Contents', [])
            for obj in contents:
                file_info = {
                    "FileName": obj['Key'],
                    "Size_Bytes": obj['Size'],
                    "LastModified": str(obj['LastModified'])
                }
                file_list.append(file_info)
                print(f"Found file: {obj['Key']} ({obj['Size']} bytes)")
        
        if not file_list:
            print("The bucket is currently empty.")
            
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': f'Successfully retrieved files from {TARGET_BUCKET}!',
                'total_files': len(file_list),
                'files': file_list
            }, indent=2)
        }
        
    except Exception as e:
        print(f"Error reading bucket objects: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }


Deleted files which are created 30 minutes before on the S3 bucket:  
<img width="975" height="542" alt="image" src="https://github.com/user-attachments/assets/98eda051-f0af-45df-bab0-02ce7418b912" />


Code: import json
import boto3
from datetime import datetime, timedelta, timezone

# Initialize the S3 client
s3 = boto3.client('s3')

def lambda_handler(event, context):
    # !!! REPLACE THIS WITH YOUR ACTUAL S3 BUCKET NAME !!!
    TARGET_BUCKET = "senthil-bucket-18-07-2026-new" 
    
    try:
        # Calculate the 30-minute expiration cutoff time (Timezone-Aware UTC)
        current_time = datetime.now(timezone.utc)
        time_cutoff = current_time - timedelta(minutes=30)
        
        print(f"Current UTC Time: {current_time}")
        print(f"Cutoff Time (Deleting anything older than): {time_cutoff}")
        
        objects_to_delete = []
        deleted_files_details = []
        
        # Paginate through the bucket to evaluate all objects securely
        paginator = s3.get_paginator('list_objects_v2')
        page_iterator = paginator.paginate(Bucket=TARGET_BUCKET)
        
        for page in page_iterator:
            contents = page.get('Contents', [])
            for obj in contents:
                # Compare file modification timestamp against the 30-minute boundary
                if obj['LastModified'] < time_cutoff:
                    # Append object identification to the bulk deletion list
                    objects_to_delete.append({'Key': obj['Key']})
                    
                    # Capture full tracking details for the execution output response
                    file_info = {
                        "FileName": obj['Key'],
                        "Size_Bytes": obj['Size'],
                        "LastModified": str(obj['LastModified'])
                    }
                    deleted_files_details.append(file_info)
                    print(f"Queued for deletion: {obj['Key']} ({obj['Size']} bytes) - Modified: {obj['LastModified']}")
        
        # Process network drop if expired files exist
        if objects_to_delete:
            total_items = len(objects_to_delete)
            print(f"Starting bulk deletion of {total_items} item(s)...")
            
            # S3 delete_objects allows dropping up to 1,000 files in a single API call
            # Chunking into groups of 1000 to manage heavy storage sets safely
            for i in range(0, total_items, 1000):
                chunk = objects_to_delete[i:i + 1000]
                s3.delete_objects(
                    Bucket=TARGET_BUCKET,
                    Delete={'Objects': chunk}
                )
            
            message = f"Successfully purged {total_items} file(s) older than 30 minutes!"
        else:
            message = "No files older than 30 minutes were found in the bucket."
            
        print(message)
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'status': 'Success',
                'message': message,
                'total_deleted_files': len(deleted_files_details),
                'deleted_files': deleted_files_details
            }, indent=2)
        }
        
    except Exception as e:
        print(f"Error executing cleanup operation: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'status': 'Error', 'error': str(e)})
        }

2. Automated EBS SnapShot creation and Cleanup:
Created Lambda function:
<img width="975" height="178" alt="image" src="https://github.com/user-attachments/assets/d8b0efcb-3a13-4aa1-b8ec-66451cc525d7" />

 
Code:
import boto3
from datetime import datetime, timedelta, timezone

# Configuration
VOLUME_ID = 'vol-0d01782bc96b4ed74'
TAG_KEY = 'CreatedBySenthil'
TAG_VALUE = 'Lambda-Backup'
RETENTION_DAYS = 30

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    now = datetime.now(timezone.utc)
    
    # 1. Create and Tag Snapshot
    try:
        snapshot = ec2.create_snapshot(
            VolumeId=VOLUME_ID,
            Description=f"Automated backup for {VOLUME_ID}"
        )
        snapshot_id = snapshot['SnapshotId']
        
        # Tag the new snapshot
        ec2.create_tags(
            Resources=[snapshot_id],
            Tags=[{'Key': TAG_KEY, 'Value': TAG_VALUE}]
        )
        print(f"Successfully created snapshot: {snapshot_id}")
    except Exception as e:
        print(f"Error creating snapshot: {e}")
        return

    # 2. Cleanup Old Snapshots
    try:
        # Filter snapshots owned by this account with the specific tag
        response = ec2.describe_snapshots(
            OwnerIds=['self'],
            Filters=[{'Name': f'tag:{TAG_KEY}', 'Values': [TAG_VALUE]}]
        )
        
        cutoff_date = now - timedelta(days=RETENTION_DAYS)
        
        for snap in response['Snapshots']:
            snap_id = snap['SnapshotId']
            snap_time = snap['StartTime'] # Boto3 returns timezone-aware UTC datetime
            
            if snap_time < cutoff_date:
                print(f"Deleting expired snapshot: {snap_id} (Created: {snap_time})")
                ec2.delete_snapshot(SnapshotId=snap_id)
                print(f"Successfully deleted snapshot: {snap_id}")
                
    except Exception as e:
        print(f"Error during snapshot cleanup: {e}")

Roles Assigned:

<img width="975" height="498" alt="image" src="https://github.com/user-attachments/assets/cbb1c91e-dbe0-4e45-8f0b-8bf39cb315c1" />

 
Snapshots created:

<img width="975" height="104" alt="image" src="https://github.com/user-attachments/assets/301501f5-f575-4b6f-86d1-8195abb5bbce" />

 

Snapshots listing using command

<img width="975" height="882" alt="image" src="https://github.com/user-attachments/assets/9464ac33-22d7-4620-9dcd-5875f80cb375" />

 

After a snapshot deletion, I am seeing only one snapshot below
<img width="975" height="35" alt="image" src="https://github.com/user-attachments/assets/da21ce44-acac-4980-b1af-c7aa3224c01a" />
<img width="975" height="555" alt="image" src="https://github.com/user-attachments/assets/6fb30c0a-2ae0-45d3-895f-00b4497aaac8" />
<img width="975" height="402" alt="image" src="https://github.com/user-attachments/assets/3eca9c82-6c8a-4d90-88cc-65a9661ae39d" />

 
 

 

Deleted snapshots which are more than 5 minutes before created
<img width="975" height="540" alt="image" src="https://github.com/user-attachments/assets/17bff842-9f61-48aa-b829-c8d1ce16780f" />





 

Logs:
<img width="975" height="364" alt="image" src="https://github.com/user-attachments/assets/8cc6e7e2-e54a-41ba-88f4-2e7d90dc63b9" />

 
Snapshot and Billing Details:

<img width="975" height="504" alt="image" src="https://github.com/user-attachments/assets/ac364297-6ba4-4947-8f3b-5e6203db3cc7" />

 
EventBridge Schedule created:
<img width="975" height="454" alt="image" src="https://github.com/user-attachments/assets/dcb6f2d9-3c46-45a1-8b17-e5fb64b3df17" />
<img width="975" height="333" alt="image" src="https://github.com/user-attachments/assets/7dd06877-6661-46d5-b69e-46ae9bd98122" />


 

 

3. Creating snapshots of an EC2

4. <img width="975" height="402" alt="image" src="https://github.com/user-attachments/assets/9dff7d62-a501-44ab-8d90-21b8317d1af9" />
<img width="975" height="463" alt="image" src="https://github.com/user-attachments/assets/a2da37f4-38b1-481b-a9f9-3998a794b9e6" />


 

 
Found the Snapshots of EC2 
5. Revert Ec2 from snapshot

<img width="975" height="128" alt="image" src="https://github.com/user-attachments/assets/b48a408d-5b95-4946-b682-feca51fbe0d5" />
<img width="975" height="607" alt="image" src="https://github.com/user-attachments/assets/bcb67a69-36b2-4887-b857-1be22a911f5a" />

 


 

6. S3-SNS-Public Access

7. <img width="975" height="428" alt="image" src="https://github.com/user-attachments/assets/cb147e5b-f30e-45be-a311-73dfad9829bb" />
<img width="975" height="502" alt="image" src="https://github.com/user-attachments/assets/7d9b5335-c351-440e-af9f-7de43f545fca" />



 

 
After enabling the Public Block it shows the below result

<img width="975" height="569" alt="image" src="https://github.com/user-attachments/assets/55483c74-8bd9-48d2-a619-08853501d418" />

 
I have fully restricted one of the s3 bucket and the other is open for public to get to know the difference.

<img width="975" height="446" alt="image" src="https://github.com/user-attachments/assets/6a0f84b0-97fb-4929-8ea2-a9669f6bf296" />

 





