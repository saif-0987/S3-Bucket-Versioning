## Ceph-S3-Bucket-Versioning

### Objectives:
HOw to enable and implement versioning in s3 to avoid accedently objects deletion.

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

3. Ensure versioning is not enabled, By default Bucket versioning in not enabled in Ceph, version ID will be null in disabled state.
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
This command also not throw any output if versioning is not enabled.
```
root@client-01:~# aws s3api get-bucket-versioning --bucket my-test-bucket-02 --endpoint-url http://192.168.9.12:8001

```

4. Enable versioning in bucket
```
root@client-01:~# aws s3api put-bucket-versioning --bucket test-bucket --versioning-configuration Status=Enabled --endpoint-url http://192.168.9.12:8001
```

5. Will verfify versioning is now enabled.
```
root@client-01:~# aws s3api get-bucket-versioning --bucket test-bucket --endpoint-url http://192.168.99.132:8001
{
    "Status": "Enabled",
    "MFADelete": "Disabled"
}
```
Also by using this command but we will see here version id is still null because we uploaded this objects before enabling vesioning
```
root@client-01:~# aws s3api list-object-versions   --bucket test-bucket --endpoint-url http://192.168.99.132:8001
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

6. Now we will upload data to see versioning is working or not
```
root@client-01:~# s3cmd put file2.txt s3://test-bucket
upload: 'file2.txt' -> 's3://test-bucket/file2.txt'  [1 of 1]
 2097152 of 2097152   100% in    0s    32.06 MB/s  done
```

7. Will verify any objects that are being uploaded after versioning enable is getting assign version id or not.
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
In above case first objects uploaded before enable , and 2nd objects that is uploaded after enable versioning.

8. Now we will delete both objects one by one and see what actually happens. First we will delete the object which are having "null" version ID.

  a. Delete `file-01.txt` file
  ```
  root@client-01:~# s3cmd rm s3://test-bucket/file-01.txt
  delete: 's3://test-bucket/file-01.txt'
  ```
  b. Verify that the file has deleted. only file2.txt is there
  ```
  root@client-01:~# s3cmd ls s3://test-bucket
  2024-09-14 08:15   2097152   s3://test-bucket/file2.txt
  ```
  c. We can see Delete Marker has been created even for this objects, keep in mind this is the object that has been uploaded to the bucket before enabling versioning.
  ```
  root@client-01:~# aws s3api list-object-versions   --bucket test-bucket --endpoint-url http://192.168.99.132:8001 --prefix file-01.txt
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

  d.Now lets try to deleet the `Delete Marker` and see if we are able to store the original objects. 
  ```
  root@client-01:~# aws s3api delete-object --bucket test-bucket --key file-01.txt --version-id a5TQiMUWerkXi3l7v0fMBxxsi84JLr. --endpoint-url http://192.168.9.12:8001
  ```
  Even after executing the command, we could not able to delete the `Delete Marker` and the this is still present and eventually we could not able to restore the objects on which the versioning was not enabled.


  9. Lets try deleting objects on which the versioning is enabled.

     a.First ensure that the object `file2.txt` is there in the bucket.
     ```
     root@client-01:~# s3cmd ls s3://test-bucket
     2024-09-14 08:15   2097152   s3://test-bucket/file2.txt
     ```
 
     b. Delete the object from the bucket.
     ```
     root@client-01:~# s3cmd rm s3://test-bucket/file2.txt
     delete: 's3://test-bucket/file2.txt'
     ```

     c. Ensure that the file is no more.
    ```
    root@client-01:~# s3cmd ls s3://test-bucket
    ```

    d. We can see dlete marker has been created against that object.
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

    e. Now we will delete the Delete marker.
    ```
    root@client-01:~# aws s3api delete-object --bucket test-bucket --key file2.txt --version-id .Jpb0cX6yB8oMYAfVH7cp9gjmXC-NNb --endpoint-url http://192.168.9.12:8001
{
    "DeleteMarker": true,
    "VersionId": ".Jpb0cX6yB8oMYAfVH7cp9gjmXC-NNb"
}
    ```

    f. Ensure the the Delete marker has been removed.
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

    g. We can now see that the objects has now been restored by following command.
    ```
    root@client-01:~# s3cmd ls s3://test-bucket
    2024-09-14 08:15   2097152   s3://test-bucket/file2.txt
    ```
    
  


