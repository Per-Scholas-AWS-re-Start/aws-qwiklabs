




create-keypair.py

```bash
#!/usr/bin/env python

import boto3

# Connect to the Amazon EC2 service
ec2_client = boto3.client('ec2')

# Create a Key Pair
key = ec2_client.create_key_pair(KeyName = 'SDK')

# Print the private Fingerprint of the private key
print(key.get('KeyFingerprint')) 
```


```bash
cleanup-keypairs.py
```

```python
#!/usr/bin/env python

import boto3

# Connect to the Amazon EC2 service
ec2_client = boto3.client('ec2')

keypairs = ec2_client.describe_key_pairs()

for key in keypairs['KeyPairs']:
  if 'lab' not in key['KeyName'].lower():
    print "Deleting key pair", key['KeyName']
    ec2_client.delete_key_pair(KeyName=key['KeyName'])
```





[ec2-user@ip-10-1-11-141 ~]$ cat snapshotter.py
#!/usr/bin/env python

import boto3
import datetime

MAX_SNAPSHOTS = 2   # Number of snapshots to keep

# Connect to the Amazon EC2 service
ec2 = boto3.resource('ec2')

# Loop through each volume
for volume in ec2.volumes.all():

  # Create a snapshot of the volume with the current time as a Description
  new_snapshot = volume.create_snapshot(Description = str(datetime.datetime.now()))
  print ("Created snapshot " + new_snapshot.id)
  
  # Too many snapshots?
  snapshots = list(volume.snapshots.all())
  if len(snapshots) > MAX_SNAPSHOTS:
    
    # Delete oldest snapshots, but keep MAX_SNAPSHOTS available
    snapshots_sorted = sorted([(s, s.start_time) for s in snapshots], key=lambda k: k[1])
    for snapshot in snapshots_sorted[:-MAX_SNAPSHOTS]:
      print ("Deleted snapshot " + snapshot[0].id)
      snapshot[0].delete()
      
## Task 5: Automate Bastion Security
      
[ec2-user@ip-10-1-11-141 ~]$ cat bastion-open
IP=`curl -s checkip.amazonaws.com`
SECURITY_GROUP_ID=`aws ec2 describe-security-groups --filters Name=group-name,Values=Bastion --query SecurityGroups[*].GroupId --output text`
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP_ID --protocol tcp --port 22 --cidr $IP/32


 [ec2-user@ip-10-1-11-141 ~]$ cat bastion-close.py
#!/usr/bin/env python

import boto3

GROUP_NAME = "Bastion"

# Connect to the Amazon EC2 service
ec2 = boto3.resource('ec2')

# Retrieve the security group
security_groups = ec2.security_groups.filter(Filters=[{'Name':'group-name', 'Values':['Bastion']}])

# Delete all rules in the group
for group in security_groups:
    group.revoke_ingress(IpPermissions = group.ip_permissions)





```bash
aws ec2 create-snapshot --description CLI --volume-id YOUR-VOLUME-ID
```
