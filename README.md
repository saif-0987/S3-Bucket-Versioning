## Enabling and Implementing S3 Bucket Versioning in Ceph

### Objectives:
Learn how to enable and implement versioning in S3 to avoid accidental deletion of objects.

### Prerequisites:
- A running Ceph Cluster.
- RGW services
- An S3 user created.
- Root-level access to the Ceph cluster.

### Procedure:

1. Create a bucket
```
root@client-01:~# s3cmd mb s3://test-bucket
Bucket 's3://test-bucket/' created
```

2. Upload data into the bucket.
```
root@client-01:~# s3cmd put file-01.txt s3://test-bucket
upload: 'file-01.txt' -> 's3://test-bucket/file-01.txt'  [1 of 1]
 1048576 of 1048576   100% in    0s    26.87 MB/s  done
```

3. Verify Versioning Status by executing below command. By default, bucket versioning is not enabled in Ceph. The version ID will be null in the disabled state.
```
root@client-01:~# aws s3api list-object-versions   --bucket test-bucket --endpoint-url http://192.168.9.12:8001
{
    "Versions": [
        {
            "ETag": "\"b6d81b360a5672d80c27430f39153e2c\"",
            "Size": 1048576,
            "StorageClass": "STANDARD",
            "Key": "file-01.txt",
            "VersionId": "null",
            "IsLatest": true,
            "LastModified": "2024-09-14T08:08:36.319000+00:00",
            "Owner": {
                "DisplayName": "test-01-user",
                "ID": "test-01-user"
            }
        }
    ],
    "RequestCharged": null
}
```
The following command will not produce any output if versioning is not enabled:
```
root@client-01:~# aws s3api get-bucket-versioning --bucket my-test-bucket-02 --endpoint-url http://192.168.9.12:8001

```

4. Enable versioning on the bucket
```
root@client-01:~# aws s3api put-bucket-versioning --bucket test-bucket --versioning-configuration Status=Enabled --endpoint-url http://192.168.9.12:8001
```

5. Verify that Versioning is Enabled
```
root@client-01:~# aws s3api get-bucket-versioning --bucket test-bucket --endpoint-url http://192.168.9.12:8001
{
    "Status": "Enabled",
    "MFADelete": "Disabled"
}
```
Note that objects uploaded before enable versioning will have a null version ID.
```
root@client-01:~# aws s3api list-object-versions   --bucket test-bucket --endpoint-url http://192.168.9.12:8001
{
    "Versions": [
        {
            "ETag": "\"b6d81b360a5672d80c27430f39153e2c\"",
            "Size": 1048576,
            "StorageClass": "STANDARD",
            "Key": "file-01.txt",
            "VersionId": "null",
            "IsLatest": true,
            "LastModified": "2024-09-14T08:08:36.319000+00:00",
            "Owner": {
                "DisplayName": "test-01-user",
                "ID": "test-01-user"
            }
        }
    ],
    "RequestCharged": null
}
```

6. Upload new data to verify versioning
```
root@client-01:~# s3cmd put file2.txt s3://test-bucket
upload: 'file2.txt' -> 's3://test-bucket/file2.txt'  [1 of 1]
 2097152 of 2097152   100% in    0s    32.06 MB/s  done
```

7. Verify version IDs for newly uploaded objects
```
root@client-01:~# aws s3api list-object-versions   --bucket test-bucket --endpoint-url http://192.168.9.12:8001
{
    "Versions": [
        {
            "ETag": "\"b6d81b360a5672d80c27430f39153e2c\"",
            "Size": 1048576,
            "StorageClass": "STANDARD",
            "Key": "file-01.txt",
            "VersionId": "null",
            "IsLatest": true,
            "LastModified": "2024-09-14T08:08:36.319000+00:00",
            "Owner": {
                "DisplayName": "test-01-user",
                "ID": "test-01-user"
            }
        },
        {
            "ETag": "\"b2d1236c286a3c0704224fe4105eca49\"",
            "Size": 2097152,
            "StorageClass": "STANDARD",
            "Key": "file2.txt",
            "VersionId": "BdqQiTNJWNcnnv3Wne4SzILcoBRxq1G",
            "IsLatest": true,
            "LastModified": "2024-09-14T08:15:38.403000+00:00",
            "Owner": {
                "DisplayName": "test-01-user",
                "ID": "test-01-user"
            }
        }
    ],
    "RequestCharged": null
}
```
Here, the first object (file-01.txt) was uploaded before enabling versioning, and the second object (file2.txt) was uploaded after versioning was enabled.

8. Delete objects and verify behavior

  a. Delete `file-01.txt` file
  ```
  root@client-01:~# s3cmd rm s3://test-bucket/file-01.txt
  delete: 's3://test-bucket/file-01.txt'
  ```
  b. Verify the delition
  ```
  root@client-01:~# s3cmd ls s3://test-bucket
  2024-09-14 08:15   2097152   s3://test-bucket/file2.txt
  ```
  c. Observe the delete marker creation
  ```
  root@client-01:~# aws s3api list-object-versions   --bucket test-bucket --endpoint-url http://192.168.9.12:8001 --prefix file-01.txt
{
    "Versions": [
        {
            "ETag": "\"b6d81b360a5672d80c27430f39153e2c\"",
            "Size": 1048576,
            "StorageClass": "STANDARD",
            "Key": "file-01.txt",
            "VersionId": "null",
            "IsLatest": false,
            "LastModified": "2024-09-14T08:08:36.319000+00:00",
            "Owner": {
                "DisplayName": "test-01",
                "ID": "test-01"
            }
        }
    ],
    "DeleteMarkers": [
        {
            "Owner": {
                "DisplayName": "test-01",
                "ID": "test-01"
            },
            "Key": "file-01.txt",
            "VersionId": "a5TQiMUWerkXi3l7v0fMBxxsi84JLr.",
            "IsLatest": true,
            "LastModified": "2024-09-14T08:55:41.415000+00:00"
        }
    ],
    "RequestCharged": null
}
   ```

  d.Attempt to delete the DeleteMarkers
  ```
  root@client-01:~# aws s3api delete-object --bucket test-bucket --key file-01.txt --version-id a5TQiMUWerkXi3l7v0fMBxxsi84JLr. --endpoint-url http://192.168.9.12:8001
  ```
  Even after executing this command, the delete marker remains, and the object cannot be restored because it was uploaded before versioning was enabled.


9. Now delete objects with versioning enabled

a.Ensure Object file2.txt exists
```
root@client-01:~# s3cmd ls s3://test-bucket
2024-09-14 08:15   2097152   s3://test-bucket/file2.txt
```
 
b. Delete the object.
```
root@client-01:~# s3cmd rm s3://test-bucket/file2.txt
delete: 's3://test-bucket/file2.txt'
```    
c. Ensure the file is no longer present
```
root@client-01:~# s3cmd ls s3://test-bucket
```

d. Observe the creation of a DeleteMarkers
```
root@client-01:~# aws s3api list-object-versions   --bucket test-bucket --endpoint-url http://192.168.9.12:8001 --prefix file2.txt
    {
    "Versions": [
        {
            "ETag": "\"b2d1236c286a3c0704224fe4105eca49\"",
            "Size": 2097152,
            "StorageClass": "STANDARD",
            "Key": "file2.txt",
            "VersionId": "BdqQiTNJWNcnnv3Wne4SzILcoBRxq1G",
            "IsLatest": false,
            "LastModified": "2024-09-14T08:15:38.403000+00:00",
            "Owner": {
                "DisplayName": "test-01",
                "ID": "test-01"
            }
        }
    ],
    "DeleteMarkers": [
        {
            "Owner": {
                "DisplayName": "test-01",
                "ID": "test-01"
            },
            "Key": "file2.txt",
            "VersionId": ".Jpb0cX6yB8oMYAfVH7cp9gjmXC-NNb",
            "IsLatest": true,
            "LastModified": "2024-09-14T09:06:36.115000+00:00"
        }
    ],
    "RequestCharged": null
}    

```

e. Now we will delete the Deletemarkers.
```
    root@client-01:~# aws s3api delete-object --bucket test-bucket --key file2.txt --version-id .Jpb0cX6yB8oMYAfVH7cp9gjmXC-NNb --endpoint-url http://192.168.9.12:8001
{
    "DeleteMarker": true,
    "VersionId": ".Jpb0cX6yB8oMYAfVH7cp9gjmXC-NNb"
}
    ```

    f. Verify that the DeleteMarker has been removed
    ```
    root@client-01:~# aws s3api list-object-versions   --bucket test-bucket --endpoint-url http://192.168.9.12:8001 --prefix file2.txt
{
    "Versions": 
    {
            "ETag": "\"b2d1236c286a3c0704224fe4105eca49\"",
            "Size": 2097152,
            "StorageClass": "STANDARD",
            "Key": "file2.txt",
            "VersionId": "BdqQiTNJWNcnnv3Wne4SzILcoBRxq1G",
            "IsLatest": true,
            "LastModified": "2024-09-14T08:15:38.403000+00:00",
            "Owner": {
                "DisplayName": "test-01",
                "ID": "test-01"
            }
        }
    ],
    "RequestCharged": null
}
```
g. Verify that the bbject has been restored
```
root@client-01:~# s3cmd ls s3://test-bucket
2024-09-14 08:15   2097152   s3://test-bucket/file2.txt
```
    
  


