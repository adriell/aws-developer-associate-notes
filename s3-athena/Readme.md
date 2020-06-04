## **AWS Fundamentals Part 4: S3**

* Buckets are defined at the region level
* Must have globally unique names
* Objects (files) have a Key. The key is the ***FULL*** path:
  * <my_bucket>/my_file.txt
  * <my_bucket>/my_folder1/another_folder/my_file.txt
* There's no concept of "directories" within buckets (althought the UI might make it appear like there is), just keys with very long names that contain slashes ("/")
* Object Values are the content of the body:
  * Max Size is 5TB
  * If uploading more than 5GB, must use "multi-part upload"
* Objects come with Metadata (list of text key / value pairs - system or user metadata)
* Tags (Unicode key / value pair - up to 10) - useful for security / lifecycle
* Version ID (if versioning is enabled)

### AWS S3 - Versioning

* Versioning is enabled at the bucket level
* It's best practice to version your buckets
  * Protect against unintended deletes
  * Easy roll back to previous version
* Any file not versioned prior to enabling versioning will have version "null"

### S3 Encryption

* There are 4 methods of encrypting objects in S3:
  * SSE-S3: encrypts S3 objects using keys handled & managed by AWS
    * Object is encrypted server side
    * AES-256 encryption type
    * Must set header: "x-amz-server-side-encryption":"AES256"
  * SSE-KMS: leverage AWS KMS to manage encryption (can use CMK)
    * Advantages: user control + audit trail
    * Object is encrypted server side
    * Must set header: "x-amz-server-side-encryption":"aws:kms"
  * SSE-C: when you use your own keys to encrypt data
    * Amazon S3 does not store the encryption key you provide
    * **HTTPS must be used**
    * Encryption key must be provided in HTTP headers, for every HTTP request made
  * Client Side Encryption: data is encrypted client side
    * Client use a library such as the Amazon S3 Encryption Client
    * Clients must encrypt data themselves before sending to S3
    * Clients must decrypt data themselves when retrieving from S3
    * Customer fully manages the key and encryption cycle
* Encryption in transit (SSL):
  * AWS S3 exposes:
    * HTTP endpoint: non encrypted
    * HTTPS endpoint: encryption in flight
  * You're able to use the endpoint you want, but HTTPS is reccomended
  * HTTPS is mandatory for SSE-C
  * Encryption in flight is also called SSL / TLS

* ### S3 Security & Bucket policies

* User based:
  * IAM policies - which API calls should be allowed for a specific user from IAM console
* Resource based:
  * Bucket Policies - bucket wide rules from S3 console - allows cross account
  * Object Access Control List (ACL) - finer grain
  * Bucket Access Control List (ACL) - less common

### S3 Bucket Policies

* JSON based policies
  * Resources: buckets and objects
  * Actions: Set of API to Allow or Deny
  * Effect: Allow / Deny
  * Principal: The account or user to apply the policy to
* We can use S3 bucket policy to:
  * Grant public access to the bucket
  * Force objects to be encrypted at upload
  * Grant access to another account (Cross Account)

### S3 Security

* Networking:
  * Supports VPC Endpoints (for instances in VPC without internet)
* Logging and Audit:
  * S3 access logs can be stored in other S3 bucket
  * API calls can be logged in AWS CloudTrail
* User Security:
  * MFA can be required in versioned buckets to delete objects
  * Signed URLs: URLs that are valid only for a  limited time

### S3 Websites

* If you get a 403 (Forbidden) error, check if the bucket policy allows public reads

### S3 CORS

* Cross Origin Resource Sharing allows you to limit the number of websites that can request your files in S3 (and limit your costs)
* If you request data from another S3 bucket, you need to enable CORS

### S3 - Consistency Model

* **Read after write consistency for PUTS of new objects**
  * As soon as an object is written, it can be retrieved.
  ex: (PUT 200 -> GET 200)
  * The exception is that if a GET is done before the object exists, the result will be cached and after writing the object it will be *eventually consistent*
  ex: (GET 404 -> PUT 200 -> GET 404)
* **Eventual Consistency for DELETES and PUTS of existing objects**
  * If we read an object after updating, we might get the older version
  * If we delete an object, we might still be able to retrieve it for a short time.

### S3 - Performance

* Before July 17th 2018
  * When you had > 100 TPS (transactions per second) S3 performance could degrade
  * Behind the curtains, each object goes to an S3 partition and for the best performance, we want the highest partition distribution
  * ***In the exam and in the past:*** It is reccomended to have random prefix in front of your key name to optimize perfomrance (never rerccomended to use date prefixes, as the pertitions would be very similar)
* After July 17th 2018 (Current state, but not in exam)
  * We can scale up to 3500 RPS for PUT and 5500 RPS for GET for EACH PREFIX
  * For reference: <https://aws.amazon.com/about-aws/whats-new/2018/07/amazon-s3-announces-increased-request-rate-performance/>
  * As stated in the news: 'This S3 request rate performance increase removes any previous guidance to randomize object prefixes to achieve faster performance.'
* Faster upload of large objects (>=100mb), use multipart upload:
  * Parallelizes PUTs for greater throughput
  * Maximize your network bandwidth and efficiency
  * Decrease time to retry in case a part fails
  * MUST use a multi-part if object size is greater than 5GB
* Use CloudFront to cache S3 objects around the world
* S3 Transfer Acceleration (uses edge locations) - just needs to change the endpoint you write to, not the code
* If using SS3-KMS encryption, you may be limited to your AWS limits for KMS usage (~100s - 1000s downlaods / uploads per second)

### S3 & Glacier Select

* If you retrieve data in S3 and Glacier, you may only want a **subset** of it
* If you retrieve all the data, the network cost may be high
* WIth S3 Select / Glacier Select, you can use SQL **SELECT** queries to let S3 or Glacier know exactly which attributes / filters you want
  * select * from s3object s where s.\"Country (Name)\" like %United States&'
* Save cost up to 80% and increase performance by up to 400%
* The "SELECT" happens "within" S3 or Glacier
* Works with files in CSV, JSON or Parquet format
* Files can be compressed with GZIP or BZIP2
* No subqueries or Joins are supported