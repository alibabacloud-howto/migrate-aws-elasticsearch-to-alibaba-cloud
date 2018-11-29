Abstract
========

>   In this document, you can find technical solution of ElasticSearch index
>   migration from AWS to Alibaba Cloud.

>   Reference architecture diagram is showed below:

![](media/13da02e1397c4873ac3eca589cf8ad35.tiff)

Introduction
============

Basic Concept
-------------

-   Elasticsearch

>   A distributed, RESTful search and analytics engine capable of solving a
>   growing number of use cases. As the heart of the Elastic Stack, it centrally
>   stores your data so you can discover the expected and uncover the
>   unexpected.

-   Kibana

>   Let’s you visualize your Elasticsearch data and navigate the Elastic Stack.

-   Amazon Elasticsearch Service

>   It’s easy to deploy, secure, operate, and scale Elasticsearch for log
>   analytics, full text search, application monitoring, and more. Amazon
>   Elasticsearch Service is a fully managed service that delivers
>   Elasticsearch’s easy-to-use APIs and real-time analytics capabilities
>   alongside the availability, scalability, and security that production
>   workloads require. 

-   Alibaba Elasticsearch Service

>   Have not onboard on International site. In this guidebook, target instance
>   is located in Alibaba Cloud China Site.

-   Snapshot and Restore

>   You can store snapshots of individual indices or an entire cluster in a
>   remote repository like a shared file system, S3, or HDFS. These snapshots
>   are great for backups because they can be restored relatively quickly.
>   However, snapshots can only be restored to versions of Elasticsearch that
>   can read the indices:

-   A snapshot of an index created in 5.x can be restored to 6.x.

-   A snapshot of an index created in 2.x can be restored to 5.x.

-   A snapshot of an index created in 1.x can be restored to 2.x.

>   Conversely, snapshots of indices created in 1.x **cannot** be restored to
>   5.x or 6.x, and snapshots of indices created in 2.x **cannot** be restored
>   to 6.x.

>   Snapshots are incremental and can contain indices created in various
>   versions of Elasticsearch. If any indices in a snapshot were created in an
>   incompatible version, you will not be able restore the snapshot.

Solution Overview
-----------------

>   ElasticSearch indices can be migrated with following steps:

>   **Create Baseline indices**

-   Create a snapshot repository and associate it to an AWS S3 Bucket

-   Create the first snapshot of the indices to be migrated, which is a full
    snapshot, the snapshot will be stored in AWS S3 bucket created in the first
    step automatically.

-   Create OSS Bucket on Alibaba Cloud, and register it to some snapshot
    repository of Alibaba Cloud ElasticSearch instance.

-   Use OSSImport tool to pull the data from AWS S3 bucket to Alibaba Cloud OSS
    bucket.

-   Restore this full snapshot to Alibaba Cloud ES instance.

>   **Periodic Incremental Snapshots**

-   Repeat serval incremental snapshot and restore.

>   **Final snapshot and Service Switchover**

-   Stop the service which can modify index data

-   Create final incremental snapshot of AWS ES instance.

-   Transfer&Restore the final incremental snapshot to Alibaba Cloud ES
    instance.

-   Service switchovers to Alibaba Cloud ES instance.

Pre-Requisites 
===============

ElasticSearch service
---------------------

>   Version number of **AWS ES is 5.5.2**, located in the region Singapore.

>   Version number of **Alibaba Cloud ES is 5.5.3**(the only version provided
>   now), located in Hangzhou.

>   The demo index name is **movies.**

Manual Snapshot Prerequisites on AWS
------------------------------------

>   Amazon ES takes daily automated snapshots of the primary index shards in a
>   domain, it stores these automated snapshots in a preconfigured Amazon S3
>   bucket for 14 days at no additional charge to you. You can use these
>   snapshots to restore the domain.

>   You cannot, however, use automated snapshots to migrate to new domains.
>   Automated snapshots are read-only from within a given domain. **For
>   migrations, you must use manual snapshots stored in your own repository (an
>   S3 bucket)**. Standard S3 charges apply to manual snapshots.

>   To create and restore index snapshots manually, you must work with IAM and
>   Amazon S3. Verify that you have met the following prerequisites before you
>   attempt to take a snapshot.

| **Prerequisite** | **Description**                                                                                                                                                                                                                                                                                                                   |
|------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| S3 bucket        | Stores manual snapshots for your Amazon ES domain.                                                                                                                                                                                                                                                                                |
| IAM role         | Delegates permissions to Amazon Elasticsearch Service. The trust relationship for the role must specify Amazon Elasticsearch Service in the Principal statement. The IAM role also is required to register your snapshot repository with Amazon ES. Only IAM users with access to this role may register the snapshot repository. |
| IAM policy       | Specifies the actions that Amazon S3 may perform with your S3 bucket. The policy must be attached to the IAM role that delegates permissions to Amazon Elasticsearch Service. The policy must specify an S3 bucket in a Resource statement.                                                                                       |

### S3 Bucket

>   You need an S3 bucket to store manual snapshots. Make a note of its Amazon
>   Resource Name (ARN). You need it for the following:

-   Resource statement of the IAM policy that is attached to your IAM role

-   Python client that is used to register a snapshot repository

>   The following example shows an ARN for an S3 bucket:

>   arn:aws:s3:::eric-es-index-backups

### IAM Role

>   You must have a role that specifies Amazon Elasticsearch
>   Service, es.amazonaws.com, in a Servicestatement in its trust relationship,
>   as shown in the following example:

>   *{*

>   *"Version": "2012-10-17",*

>   *"Statement": [*

>   *{*

>   *"Sid": "",*

>   *"Effect": "Allow",*

>   *"Principal": {*

>   *"Service": "es.amazonaws.com"*

>   *},*

>   *"Action": "sts:AssumeRole"*

>   *}*

>   *]*

>   *}*

>   In AWS IAM Console, you can find **Trust Relationship** deital here:

![](media/8294d97a833740011ef02621d9b8148a.tiff)

![](media/f94af72e3a91cbf2f6a3a10e95fd411f.tiff)

>   When you create an AWS service role using the IAM console, Amazon ES is not
>   included in the **Select role type** list. However, you can still create the
>   role by choosing **Amazon EC2**, following the steps to create the role, and
>   then editing the role's trust relationships to es.amazonaws.com instead
>   of ec2.amazonaws.com. 

### IAM Policy

>   You must attach an **IAM policy** to the **IAM role**. The policy specifies
>   the S3 bucket that is used to store manual snapshots for your Amazon ES
>   domain. The following example specifies the ARN of
>   the eric-es-index-backups bucket:

>   *{*

>   *"Version": "2012-10-17",*

>   *"Statement": [*

>   *{*

>   *"Action": [*

>   *"s3:ListBucket"*

>   *],*

>   *"Effect": "Allow",*

>   *"Resource": [*

>   *"arn:aws:s3:::eric-es-index-backups"*

>   *]*

>   *},*

>   *{*

>   *"Action": [*

>   *"s3:GetObject",*

>   *"s3:PutObject",*

>   *"s3:DeleteObject"*

>   *],*

>   *"Effect": "Allow",*

>   *"Resource": [*

>   *"arn:aws:s3:::eric-es-index-backups/\*"*

>   *]*

>   *}*

>   *]*

>   *}*

>   You need paste it here:

![](media/18997183f04a7760d3af3c0d3a1538c0.tiff)

>   Another readable view:

![](media/11883dc7d988bdbf91cea0a0f282e4f8.tiff)

### Attach IAM Policy to IAM Role

![](media/84f1c5545df547e1d118e73db10e9505.tiff)

Registering a Manual Snapshot Directory
=======================================

>   You must register the snapshot directory with Amazon Elasticsearch Service
>   before you can take manual index snapshots. This one-time operation requires
>   that you sign your AWS request with credentials for one of the users or
>   roles specified in the IAM role's trust relationship, as described in
>   chapter 3.2

>   You can't use curl to perform this operation because it doesn't support AWS
>   request signing. Instead, use the sample Python client to register your
>   snapshot directory.

Modify Sample Python Client
---------------------------

>   Modify the values with background color yellow to your real value in
>   attachment document, then copy content of attachment into a python file
>   **snapshot.py** after customization.

| **Variable name**     | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
|-----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| region                | AWS Region where you created the snapshot repository                                                                                                                                                                                                                                                                                                                                                                                                                            |
| host                  | Endpoint for your Amazon ES domain                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| aws_access_key_id     | IAM credential                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| aws_secret_access_key | IAM credential                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| path                  | Name of the snapshot repository                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| data                  | Must include the name of the S3 bucket and the ARN for the IAM role that you created in chapter 3.2. To enable [server-side encryption with S3-managed keys](http://docs.aws.amazon.com/AmazonS3/latest/dev/UsingServerSideEncryption.html) for the snapshot repository, add "server_side_encryption": true to the "settings" JSON. **Important** If the S3 bucket is in the us-east-1 region, you need to use "endpoint": "s3.amazonaws.com"in place of "region": "us-east-1". |

-   bucket

-   region

-   role_arn

Install Amazon Web Services Library boto-2.48.0
-----------------------------------------------

>   This sample Python client requires that you install version 2.*x* of
>   the boto package on the computer where you register your snapshot
>   repository.

>   *\# wget*
>   <https://pypi.python.org/packages/66/e7/fe1db6a5ed53831b53b8a6695a8f134a58833cadb5f2740802bc3730ac15/boto-2.48.0.tar.gz#md5=ce4589dd9c1d7f5d347363223ae1b970>

>   *\# tar zxvf boto-2.48.0.tar.gz*

>   *\# cd boto-2.48.0*

>   *\# python setup.py install*

Execute Python Client to Register Snapshot Directory
----------------------------------------------------

>   *\# python snapshot.py*

>   Registering Snapshot Repository

>   {"acknowledged":true}

>   Check result in **KibanaDev Tools** with request：

>   *GET \_snapshot*

![](media/a48b94be7b609e7b82777ce00aabea5f.tiff)

Snapshot and Restore for the first time
=======================================

Take a snapshot manually on AWS ES
----------------------------------

>   Following commands are all performed on **KibanaDevTools**, you can also
>   perform them in the format of **curl** command in Linux.

-   Take a snapshot with name **snapshot_moives_1** only for the index
    **movies** in the repository **eric-snapshot-repository**.

>   *PUT \_snapshot/eric-snapshot-repository/snapshot_movies_1*

>   *{*

>   *"indices": "movies"*

>   *}*

-   Check snapshot status

>   *GET \_snapshot/ eric-snapshot-repository/snapshot_movies_1*

![](media/dc2081f678ba0cbc5fb6f5324e9310ff.tiff)

>   Check snapshot files on the AWS S3 console:

![](media/c1688f6618baf43f0a24ba45443ffc13.tiff)

Pull snapshot data from AWS S3 to Alibaba Cloud OSS
---------------------------------------------------

>   In this step, you need pull snapshot data from AWS S3 bucket to Alibaba
>   Cloud OSS.

>   Please refer to [Migrate data from Amazon S3 to Alibaba Cloud
>   OSS](https://www.alibabacloud.com/help/doc-detail/64919.htm?spm=a3c0i.l31815en.b99.105.67c25139Y3tgnc)

>   After data transfer, check stored snapshot data on OSS console:

![](media/fb01981a005d9528de7779d3171dcf48.tiff)

1.  Restore snapshot to Alibaba Cloud ES instance

    1.  Create Snapshot Repository

>   Perform following request on **KibanaDev Tools** to create snapshot
>   repository with same name, modify values with red color to your real ones.

>   *PUT \_snapshot/eric-snapshot-repository*

>   *{*

>   *"type": "oss",*

>   *"settings": {*

>   *"endpoint": "http://oss-cn-hangzhou-internal.aliyuncs.com",*

>   *"access_key_id": "Put your access key id here.",*

>   *"secret_access_key": "Put your secret access key here.",*

>   *"bucket": "eric-oss-aws-es-snapshot-s3",*

>   *"compress": true*

>   *}*

>   *}*

![](media/1514294f52d7b1852db6f94d4e695a93.tiff)

>   After creating the snapshot directory, check the snapshot status with
>   snapshot name **snapshot_movies_1** which is assigned in AWS ES manual
>   snapshot step.

>   *GET \_snapshot/eric-snapshot-repository/snapshot_movies_1*

![](media/c9a3d4d113928c07065a40925dbdf451.tiff)

>   *\*\* Please record the start/end time of this snapshot operation, it will
>   be used in future incremental snapshot data transfer by Alibaba Cloud
>   OSSimport tool.*

>   *"start_time_in_millis": 1519786844591*

>   *"end_time_in_millis": 1519786846236,*

### Restore snapshots

>   Perform following request on **KibanaDev Tools**

>   *POST \_snapshot/eric-snapshot-repository/snapshot_movies_1/_restore*

>   *{*

>   *"indices": "movies"*

>   *}*

>   *GET movies/_recovery*

![](media/6f7c5c53b58552614370959c779b2fca.tiff)

>   Check the availability of index movies on **KibanaDiscover**, we can see
>   there exists 3 records in the index movies, as same as AWS ES instance.

![](media/0a9f4d8e16615b42daba8299f02a1e03.tiff)

Snapshot and Restore for the last time
======================================

Create some sample data on AWS ES index movies
----------------------------------------------

>   From above, we know that there are only 3 records in the index movies, so we
>   insert another 2 records.

![](media/2f8f2e58d03d03851839a91a9116c796.tiff)

>   or you can use request: *GET movies/_count*

Take another snapshot manually
------------------------------

>   Refer to chapter 5.1, then check the snapshot status:

![](media/b0a06a3bf4188169e7eb401012ecec32.tiff)

>   Check files on S3 bucket:

![](media/4bcf3dff243dc5d5f8df452be4651515.tiff)

>   If you check the folder **indices**, you can also find some differences.

Pull incremental snapshot data from AWS S3 to Alibaba Cloud OSS
---------------------------------------------------------------

>   You can also use OSSImport tool to migrate data from S3 to OSS, because
>   there are 2 snapshots files stored on S3 bucket now, so we try to migrate
>   difference data via modify the value of filed **isSkipExistFile** in the
>   configuration file **local_job.cfg**

| Filed           | Meaning                                                                      | Description                                                                                   |
|-----------------|------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| isSkipExistFile | Whether to skip the existing objects during data migration, a Boolean value. | If it is set to **true**, the objects are skipped according to the size and LastModifiedTime. |

-   If it is set to **false**, the existing objects are overwritten. The default
    value is false. This option is invalid when jobType is set to audit.

>   After OSSImport migration job complete, we can find only ‘new’ files are
>   migrated to OSS.

-   On Alibaba Cloud OSS bucket

![](media/887a114bb4cd33529c07db76668515bb.tiff)

-   On AWS S3 bucket

![](media/2b32ee7b79b1902c5d7e2c609a22d61b.tiff)

Restore incremental snapshot
----------------------------

>   Can also follow the steps in the chapter 5.3.2, but index movies is needed
>   to be close first, then restore the snapshot, open again after the restore.

>   *POST /movies/_close*

>   *GET movies/_stats*

>   *POST \_snapshot/eric-snapshot-repository/snapshot_movies_2/_restore*

>   *{*

>   *"indices": "movies"*

>   *}*

>   *POST /movies/_open*

>   After restore procedure complete, we can see the count(5) of documents in
>   the index movies is as same as it is in AWS ES instance.

![](media/a6d84d2dde16ed06b0e16ca4e840d3c3.tiff)

Conclusion
==========

>   It’s possible to migrate AWS Elasticsearch service data to Alibaba Cloud
>   Elasticsearch service via snapshot and restore method.

>   This migration technology solution requires AWS Elasticsearch instance to
>   stop writing business requests during the migration and recovery of the last
>   snapshot.

Further Reading
===============

-   *https://www.elastic.co/products/elasticsearch*

-   <https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html>

-   <https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-managedomains-snapshots.html>

-   <https://www.alibabacloud.com/help/doc-detail/64919.htm?spm=a3c0i.l31815en.b99.105.67c25139Y3tgnc>

-   <https://github.com/zhichen/elasticsearch-repository-oss/wiki/OSS%E5%BF%AB%E7%85%A7%E8%BF%81%E7%A7%BB?spm=a2c4g.11186623.2.3.2acd85>

-   <https://github.com/zhichen/elasticsearch-repository-oss/wiki/OSS%E5%BF%AB%E7%85%A7%E8%BF%81%E7%A7%BB?spm=a2c4g.11186623.2.3.wfCX30>

-   <https://www.alibabacloud.com/help/doc-detail/56990.htm?spm=a3c0i.o56990en.a1.4.5baa6605O1C9yZ>
