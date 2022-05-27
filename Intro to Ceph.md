# Installing Ceph

Before installation, make sure Docker is installed. You don't need to set up docker or create any containers yourself. Ceph will manage this for you. 
Also make sure lvm2 is installed ```sudo apt install -y lvm2```. 

##Cephadm

The installation process of ceph using cephadm can be found [here](https://docs.ceph.com/en/quincy/cephadm/install/#install-cephadm).
Remark: This is the link for the version quincy. The documentation of the newest version is always accessible via [docs.ceph.com](https://docs.ceph.com/)
The ceph command can later be installed as well. Remark: this ceph command will need sudo rights. This error message when not providing this right is very vague. 

### Setting up a cluster
#### Bootstrap
The first step after installing cephadm is to run the bootstrap command. 

```sudo cephadm bootstrap --mon-ip *<mon-ip>*```

This mon-ip is the ip of the admin host. 

#### Dashboard
After running this command, the dashboard can be accessed. The login credentials will be displayed on the terminal after running the initial bootstrap command. 
You will be asked to choose a different password after the first login. 

The dashboard provides a UI to access the cluster. By default the dashboard is enabled. But if not it can be enabled using 

```sudo ceph mgr module enable dashboard```

The dashboard has SSL support, and this is enabled by default. To get access you need to provide signed certificates. SSL can be also be disabled. 
When enabled, the dashboard will bind to port 8443 of the host ip. when disabled, port 8080 is used. 

More info on the dashboard can be found [here](https://docs.ceph.com/en/quincy/mgr/dashboard/). 

#### Adding hosts ans storage

The easiest way to add the other hosts to the cluster is from within the dashboard. This provides an easy to use UI for setting up the different hosts, and you can label the different services that the host will run. 
After labeling, the different hosts will install the necessary requirements. 

Storage can be added by physically plugging in empty SSDs or HDDs to the hosts. 

```sudo ceph orch apply osd --all-available-devices```
will install the osd's to all devices. 

## Rados Gateway for s3

First the radosgw package needs to be installed on all hosts. ```sudo apt install radosgw```. 
Next, create a directory for rgw and create a keyring to allow the osd's to be read, write and executable. This needs to be done on all hosts. 

```sudo mkdir -p /var/lib/ceph/radosgw/ceph-rgw.`hostname -s`
sudo ceph auth get-or-create client.rgw.`hostname -s` osd 'allow rwx' mon 'allow rw' -o /var/lib/ceph/radosgw/ceph-rgw.`hostname -s`/keyring```

Next, update the configurartion file under ```/etc/ceph/ceph.conf``` to include:

```
[global]
    fsid = 42c82826-b1b6-11ec-9cee-5fc1dc390bd1
    mon initial members = hostX,hostY,hostZ
    mon host = IPX,IPY,IPZ

[client.rgw.hostX]
    host = hostX
    keyring = /var/lib/ceph/radosgw/ceph-rgw.hostX/keyring
    log file = /var/log/ceph/ceph-rgw-hostX.log
    rgw frontends = "beast endpoint=IPX:80"
    rgw thread pool size = 512
```
Where HostX, HostY and HostZ are the names of the hosts, and IPX, IPY and IPZ are the IPs of the respective hosts. 
This needs to be added for all hosts. Only hostX needs to be changed to HostY and HostZ in the entire client.rgw.hostX block for respective hosts. 
By default, rgw will bind to port 80 without SSL and to port 443 with SSL. 


Next, give direct ssh access to the different hosts. 
On the main host, the one where the cephadm bootstrap command was run, the public key can be found in ```/etc/ceph/ceph.pub```. 
Copy-paste the key to the other hosts. The easiest way is to paste the key in the file ```/root/.ssh/authorized_keys``` on the other hosts. For this you need root access.

To enable rgw, execute following commands.

```
sudo systemctl start ceph-radosgw@rgw.`hostname -s`
sudo systemctl status ceph-radosgw@rgw.`hostname -s`
sudo systemctl enable ceph-radosgw@rgw.`hostname -s`
```

If the status shows "active" on all hosts, rgw is running correctly. 

To create users and buckets from within the dashboard, run the command ```sudo ceph dashboard set-rgw-credentials``` to provide access from within the dashboard.

A video explaining ths setup for s3, and more on setting up zones can be found [here](https://www.youtube.com/watch?v=6uX23Q3y_SU&t=453s)

### Bucket permissions
Read and Write permissions can be set for buckets for specific users. 
When creating a bucket, an owner needs to be selected. This owner will always have read and write access. 

For adding a policy for a specific bucket, start by creating a file, and enter the policy in that file. Here is an example of such policy:

```{
  "Version": "2012-10-17",
  "Statement": [
  {
    "Effect": "Allow",
    "Principal": {"AWS": ["arn:aws:iam:::user/USER1"]},
    "Action": ["s3:ListBucket", "s3:PutObject", "s3:GetObject"],
    "Resource": ["arn:aws:s3:::BUCKETNAME",
    "arn:aws:s3:::BUCKETNAME/*"]
  },
  {
    
    "Effect": "Allow",
    "Principal": {"AWS": ["arn:aws:iam:::user/USER2"]},
    "Action": ["s3:ListBucket", "s3:GetObject"],
    "Resource": ["arn:aws:s3:::BUCKETNAME",
    "arn:aws:s3:::BUCKETNAME/*"]
  }]
}
```

USER1, USER2 and BUCKETNAME need to be changed, as well as the actions the specific user has access to. 

To set the policy, use ```s3cmd setpolicy examplepol s3://BUCKETNAME```. 
s3cmd might need to be installed and configured separately. More info can be found [here](https://help.dreamhost.com/hc/en-us/articles/215916627-Installing-S3cmd)

More info on bucket policies and available actions can be found [here](https://docs.ceph.com/en/quincy/radosgw/bucketpolicy/).

### Accessing the data using s3

There are different methods for storing and reading the data. Operations within terminal can be accessed with ```s3cmd```. 
Another way is with scripts. Basic python scripts are created by [ronald-d-dilley-jr](https://github.com/ronald-d-dilley-jr/ceph-s3-examples) on Github. 
I extracted the most important ones:

```python
import boto3
from io import BytesIO

access_key = 'Access Key'
secret_key = 'Secret Key'
host = 'http://HOST_IP:80'

def list_buckets(s3):
    response = s3.list_buckets()
    for item in response['Buckets']:
        print(item['CreationDate'], item['Name'])
    print('--------')

def list_objects(s3, bucket_name):
    response = s3.list_objects_v2(Bucket=bucket_name)
    for item in response['Contents']:
        print(item['Key'])

def create_bucket(s3, bucket_name):
    bucket = s3.Bucket(bucket_name)
    bucket.create()

def delete_bucket(s3, bucket_name):
    bucket = s3.Bucket(bucket_name)
    bucket.delete()

def upload_file(s3, bucket_name, key_name, file_name):
    bucket = s3.Bucket(bucket_name)
    bucket.upload_file(Filename=file_name, Key=key_name)

def download_file(s3, bucket_name, key_name, file_name):
    bucket = s3.Bucket(bucket_name)
    bucket.download_file(Filename=file_name, Key=key_name)

def download_fileobj(s3, bucket_name, key_name):
    bucket = s3.Bucket(bucket_name)
    data = BytesIO()
    bucket.download_fileobj(Fileobj=data, Key=key_name)
    return data


def put_objects(s3, bucket_name, key_name, data):
    bucket = s3.Bucket(bucket_name)
    bucket.put_object(Bucket=bucket_name, Key=key_name, Body=data)

def main():
    client = boto3.client('s3',
                          endpoint_url=host,
                          aws_access_key_id=access_key,
                          aws_secret_access_key=secret_key)

    resource = boto3.resource('s3',
                              endpoint_url=host,
                              aws_access_key_id=access_key,
                              aws_secret_access_key=secret_key)

    list_buckets(client)


if __name__ == '__main__':
    main()
```

