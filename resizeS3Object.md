# Using an Amazon S3 trigger to create thumbnail images #
In this hands-on, you create a Lambda function and configure a trigger for Amazon Simple Storage Service (Amazon S3). Amazon S3 invokes the lambda function for each image file that is uploaded to an S3 bucket. The function reads the image object from the source S3 bucket and creates a thumbnail image to save in a target S3 bucket.
## Step 1. Create S3 buckets and upload a sample object ##
1. Open the [Amazon S3 console](https://console.aws.amazon.com/s3/home?region=us-east-1)
1. Create two S3 buckets. The target bucket must be named **source-resized**, where **source** is the name of the source bucket. For example, a source bucket named _mybucket_ and a target bucket named _mybucket-resized_.
1. In the source bucket, upload a .jpg object, for example, _HappyFace.jpg_
## Step 2. Create the IAM policy ##
1. Open the [Policies page](https://console.aws.amazon.com/iamv2/home#/policies) in the IAM console.
1. Choose Create policy
1. Choose the JSON tab, and then paste the following policy. Be sure to replace mybucket with the name of the source bucket that you created previously
``` {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:PutLogEvents",
                "logs:CreateLogGroup",
                "logs:CreateLogStream"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::mybucket/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::mybucket-resized/*"
        }
    ]
}

```
Choose Next: Tags,Choose Next: Review,
Under Review policy, for Name, enter **AWSLambdaS3Policy**,Choose Create policy.
## Step 3. Create the execution role ##
1. Open the [Roles page](https://console.aws.amazon.com/iamv2/home#/roles) in the IAM console.
1. Choose Create role
1. Create a role with the following properties: 
*  Trusted entity – Lambda
* Permissions policy – AWSLambdaS3Policy
*  Role name – lambda-s3-role
## Step 4. Create the function code ##
Copy the following code example into a file named index.js
```// dependencies
const AWS = require('aws-sdk');
const util = require('util');
const sharp = require('sharp');

// get reference to S3 client
const s3 = new AWS.S3();

exports.handler = async (event, context, callback) => {

  // Read options from the event parameter.
  console.log("Reading options from event:\n", util.inspect(event, {depth: 5}));
  const srcBucket = event.Records[0].s3.bucket.name;
  // Object key may have spaces or unicode non-ASCII characters.
  const srcKey    = decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, " "));
  const dstBucket = srcBucket + "-resized";
  const dstKey    = "resized-" + srcKey;

  // Infer the image type from the file suffix.
  const typeMatch = srcKey.match(/\.([^.]*)$/);
  if (!typeMatch) {
      console.log("Could not determine the image type.");
      return;
  }

  // Check that the image type is supported
  const imageType = typeMatch[1].toLowerCase();
  if (imageType != "jpg" && imageType != "png") {
      console.log(`Unsupported image type: ${imageType}`);
      return;
  }

  // Download the image from the S3 source bucket.

  try {
      const params = {
          Bucket: srcBucket,
          Key: srcKey
      };
      var origimage = await s3.getObject(params).promise();

  } catch (error) {
      console.log(error);
      return;
  }

  // set thumbnail width. Resize will set the height automatically to maintain aspect ratio.
  const width  = 200;

  // Use the sharp module to resize the image and save in a buffer.
  try {
      var buffer = await sharp(origimage.Body).resize(width).toBuffer();

  } catch (error) {
      console.log(error);
      return;
  }

  // Upload the thumbnail image to the destination bucket
  try {
      const destparams = {
          Bucket: dstBucket,
          Key: dstKey,
          Body: buffer,
          ContentType: "image"
      };

      const putResult = await s3.putObject(destparams).promise();

  } catch (error) {
      console.log(error);
      return;
  }

  console.log('Successfully resized ' + srcBucket + '/' + srcKey +
      ' and uploaded to ' + dstBucket + '/' + dstKey);
};
```
## Step 5. Create the deployment package ##
1. Launch Amazon  Linux instance with a public DNS name that is reachable from the Internet and to which you are able to connect using SSH.You must also have configured your security group to allow SSH (port 22), HTTP (port 80), and HTTPS (port 443) connections
1. Connect to your Linux instance as ec2-user using SSH
1. Install node version manager (nvm) by typing the following at the command line.
```
 curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
 ```
 ```
. ~/.nvm/nvm.sh
```
```
nvm install node

```
Test that Node.js is installed and running correctly by typing the following at the command line.
```
node -e "console.log('Running Node.js ' + process.version)"
```
Install the sharp library with npm. For Linux, use the following command:
```
npm install sharp
```
Create a deployment package with the function code and its dependencies. Set the -r (recursive) option for the zip command to compress the subfolders
```
zip -r function.zip .
```
## Step 6. Send the zip file to S3 ##
1. Gave S3 Full access to EC2 for sending file to S3 bucket
1. Create a bucket with a **Lamnda_Function** name
1. In Terminal send the zip file to s3 bucket:
```
 aws s3 cp <filename> s3://<bucket-name>
```
## Step 7. Create the Lambda function ##
1. Open the [Functions page](https://console.aws.amazon.com/lambda/home?region=us-east-1#/functions) on the Lambda console.
1. Choose Create function
1. Function name: (write any name you want)
1. Runtime: Node.js 12.x
1. Use an existing role: lambda-s3-role
1. Choose Create function.
## Step 8. Configure an Amazon S3 Bucket as a Lambda Event Source ##
1. Click Add trigger and then configure
1. Select a trigger: S3
1. Bucket: Select your bucket
1. Event type: All object create events
1. Scroll to the bottom and click “Add”
## Step 9. Configure the Lambda function ##
1. Code entry type: Upload a file from Amazon S3
(write your url from s3 which is you send your zip file)
## Step 10. Test in the console ##
Invoke the Lambda function manually using sample Amazon S3 event data.

1.  On the Code tab, under Code source, choose the arrow next to Test, and then choose Configure test events from the dropdown list
1. In the Configure test event window, do the following:
* Choose Create new test event.
* For Event template, choose Amazon S3 Put.
* For Event name, enter a name for the test event. For example, mys3testevent.
* In the test event JSON, replace the S3 bucket name (example-bucket) and object key (test/key) with your bucket name and test file name. Your test event should look similar to the following:

```
{
  "Records": [
    {
      "eventVersion": "2.0",
      "eventSource": "aws:s3",
      "awsRegion": "us-west-2",
      "eventTime": "1970-01-01T00:00:00.000Z",
      "eventName": "ObjectCreated:Put",
      "userIdentity": {
        "principalId": "EXAMPLE"
      },
      "requestParameters": {
        "sourceIPAddress": "127.0.0.1"
      },
      "responseElements": {
        "x-amz-request-id": "EXAMPLE123456789",
        "x-amz-id-2": "EXAMPLE123/5678abcdefghijklambdaisawesome/mnopqrstuvwxyzABCDEFGH"
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "testConfigRule",
        "bucket": {
          "name": "my-s3-bucket",
          "ownerIdentity": {
            "principalId": "EXAMPLE"
          },
          "arn": "arn:aws:s3:::example-bucket"
        },
        "object": {
          "key": "HappyFace.jpg",
          "size": 1024,
          "eTag": "0123456789abcdef0123456789abcdef",
          "sequencer": "0A1B2C3D4E5F678901"
        }
      }
    }
  ]
}
```
* Choose Create.
* To invoke the function with your test event, under Code source, choose Test.
## Step 11. Test with the S3 trigger ##
1. On the Buckets page of the Amazon S3 console, choose the name of the source bucket that you created earlier
1. On the Upload page, upload a few .jpg or .png image files to the bucket.
1. Verify for each image object that a thumbnail is created in the target S3 bucket using the your Lambda function.












