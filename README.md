Here's a detailed step-by-step guide on how to set up an AWS Lambda function to use AWS Elastic Transcoder for compressing videos when they are uploaded to an S3 bucket.

### Step 1: Set Up Your S3 Bucket

1. **Create an S3 Bucket**:
   - Go to the [AWS S3 Console](https://console.aws.amazon.com/s3/).
   - Click on **Create bucket**.
   - Enter a **Bucket name** and select your **Region**.
   - Configure any additional settings as needed and click **Create bucket**.

2. **Set Up Bucket Permissions**:
   - After creating your bucket, go to the **Permissions** tab.
   - Edit the **Block public access** settings if necessary, but keep your bucket secure.
   - You can also configure bucket policies if you want to allow certain actions from specific IAM roles or users.

### Step 2: Create an Elastic Transcoder Pipeline

1. **Go to the Elastic Transcoder Console**:
   - Navigate to the [Elastic Transcoder Console](https://console.aws.amazon.com/elastictranscoder/home).

2. **Create a Pipeline**:
   - Click on **Pipelines** in the left sidebar.
   - Click **Create New Pipeline**.
   - Fill in the following fields:
     - **Pipeline name**: Choose a descriptive name.
     - **Input bucket**: Select the S3 bucket you created.
     - **Output bucket**: Choose another (or the same) S3 bucket for the output.
     - **Role**: Create a new role or use an existing one. The role needs permission to read from the input bucket and write to the output bucket.
   - Click **Create Pipeline**.

3. **Note the Pipeline ID**:
   - After creation, note down the **Pipeline ID**, as you will need it in your Lambda function.

### Step 3: Create a Lambda Function

1. **Go to the Lambda Console**:
   - Navigate to the [AWS Lambda Console](https://console.aws.amazon.com/lambda/).

2. **Create a New Lambda Function**:
   - Click on **Create function**.
   - Choose **Author from scratch**.
   - Enter the **Function name**.
   - Select **Python 3.x** as the runtime.
   - For **Execution role**, choose **Create a new role with basic Lambda permissions**.
   - Click **Create function**.

3. **Set Up Lambda Function Permissions**:
   - Once your function is created, click on the **Configuration** tab.
   - Click on **Permissions** and then the role link to edit it.
   - Add permissions to allow the function to interact with Elastic Transcoder and S3.

   Here’s an example policy to add:
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": [
                   "s3:GetObject",
                   "s3:PutObject"
               ],
               "Resource": "arn:aws:s3:::your-input-bucket-name/*"
           },
           {
               "Effect": "Allow",
               "Action": "elastictranscoder:CreateJob",
               "Resource": "*"
           }
       ]
   }
   ```

### Step 4: Write the Lambda Function Code

1. **Edit the Lambda Function Code**:
   - In the Lambda console, scroll down to the **Function code** section.
   - Replace the existing code with the following:
   ```python
   import json
   import boto3

   def lambda_handler(event, context):
       # Set up Elastic Transcoder client
       transcoder = boto3.client('elastictranscoder', region_name='us-east-1')
       
       # Parse the S3 bucket and object key from the event
       s3_bucket = event['Records'][0]['s3']['bucket']['name']
       s3_key = event['Records'][0]['s3']['object']['key']

       # Elastic Transcoder pipeline ID and output settings
       pipeline_id = '1726740568534-wz1yku'  # Replace with your pipeline ID
       output_prefix = 'uploads/videos/encoded/'

       # Create the Elastic Transcoder job
       try:
           response = transcoder.create_job(
               PipelineId=pipeline_id,
               Input={
                   'Key': s3_key,
                   'FrameRate': 'auto',
                   'Resolution': 'auto',
                   'AspectRatio': 'auto',
                   'Interlaced': 'auto',
                   'Container': 'mp4'  # Specify a compatible container format
               },
               Outputs=[
                   {
                       'Key': output_prefix + s3_key.split('/')[-1],
                       'PresetId': '1351620000001-000010',  # Preset ID for 720p HD
                       'ThumbnailPattern': '',
                       'Rotate': 'auto',
                   }
               ]
           )
           print(f"Transcoding job created successfully: {response['Job']['Id']}")
       except Exception as e:
           print(f"Error creating transcoding job: {str(e)}")

       return {
           'statusCode': 200,
           'body': json.dumps('Job created successfully')
       }
   ```

2. **Replace the Pipeline ID**:
   - Update the `pipeline_id` in the code with the actual pipeline ID you noted earlier.

### Step 5: Set Up S3 Trigger for the Lambda Function

1. **Go Back to Your S3 Bucket**:
   - In the S3 console, select the bucket where you want to upload videos.

2. **Configure Event Notifications**:
   - Go to the **Properties** tab of the bucket.
   - Scroll down to **Event notifications** and click **Create event notification**.
   - Name your event, select **All object create events**, and select **Lambda Function** as the destination.
   - Choose your Lambda function from the dropdown list and save the configuration.

### Step 6: Test the Setup

1. **Upload a Test Video**:
   - Use the AWS S3 console or `aws s3 cp` command to upload a test video file to your S3 bucket.

2. **Check CloudWatch Logs**:
   - Go to the [CloudWatch Console](https://console.aws.amazon.com/cloudwatch/) and check the logs for your Lambda function to see if the job was created successfully.
   - Look for any error messages in case something went wrong.

3. **Check Output Bucket**:
   - After the Elastic Transcoder job completes, check the specified output bucket for the compressed video.

### Step 7: Monitor and Troubleshoot

- Keep an eye on the CloudWatch logs for your Lambda function to catch any errors or logs related to job creation.
- Check the Elastic Transcoder console to monitor the status of your transcoding jobs.

### Conclusion
By following these steps, you should have a functioning AWS Lambda function that automatically creates an Elastic Transcoder job whenever a video is uploaded to your S3 bucket, effectively compressing it as specified.
