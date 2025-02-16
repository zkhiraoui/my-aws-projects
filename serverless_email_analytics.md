<!DOCTYPE html>
<html>
<head>
   
</head>
<body>

<h1>🚀 Serverless Email Analytics System using AWS Services</h1>

<div class="tip">
    <strong>Project Overview:</strong> This system tracks email delivery status, bounces, and complaints using AWS SES, with automated processing and storage in S3 via Lambda functions.
</div>

<h2>📋 Prerequisites</h2>
<ul>
    <li>AWS Account with appropriate permissions</li>
    <li>AWS CLI installed and configured</li>
    <li>Basic understanding of AWS services (SES, SNS, Lambda, S3)</li>
    <li>Verified domain in SES</li>
</ul>

<h2>🛠️ Step-by-Step Implementation</h2>

<div class="step">
    <h3>STEP 1: Create S3 Bucket</h3>
    <div class="code-block">
        <pre>aws s3 mb s3://your-ses-notifications --region us-west-2</pre>
    </div>
    <p class="tip">This bucket will store all your email analytics data.</p>
</div>

<div class="step">
    <h3>STEP 2: Create SNS Topic</h3>
    <div class="code-block">
        <pre>aws sns create-topic --name ses-notifications</pre>
    </div>
    <p class="tip">This topic will receive notifications from SES.</p>
</div>

<div class="step">
    <h3>STEP 3: Create Lambda Function</h3>
    <p>Create a new file named <code>lambda_function.py</code> with the following code:</p>
    <div class="code-block">
        <!-- Your Lambda function code here -->
        <pre>import json
import boto3
import datetime

s3 = boto3.client('s3')
BUCKET_NAME = 'your-ses-notifications'

def lambda_handler(event, context):
    try:
        # Get the SNS message
        message = json.loads(event['Records'][0]['Sns']['Message'])
        
        # Extract email details
        mail = message.get('mail', {})
        message_id = mail.get('messageId', 'unknown')
        current_date = datetime.datetime.now()
        month_year = current_date.strftime('%Y-%m')
        timestamp = current_date.strftime('%Y-%m-%d %H:%M:%S')
        
        # Determine the type and create specific content
        if 'bounce' in message:
            file_key = f'bounces/bounces-{month_year}.json'
            status = {
                'type': 'bounce',
                'bounceType': message['bounce']['bounceType'],
                'bounceSubType': message['bounce']['bounceSubType'],
                'recipients': message['bounce']['bouncedRecipients']
            }
        elif 'complaint' in message:
            file_key = f'complaints/complaints-{month_year}.json'
            status = {
                'type': 'complaint',
                'complaintFeedbackType': message['complaint'].get('complaintFeedbackType', 'unknown'),
                'recipients': message['complaint']['complainedRecipients']
            }
        elif 'delivery' in message:
            file_key = f'delivered/delivered-{month_year}.json'
            status = {
                'type': 'delivery',
                'recipients': message['delivery']['recipients'],
                'timestamp': message['delivery']['timestamp']
            }
        else:
            return {
                'statusCode': 200,
                'body': 'Skipped - not a relevant notification type'
            }
        
        # Create new entry
        new_entry = {
            'messageId': message_id,
            'timestamp': timestamp,
            'status': status,
            'sourceEmail': mail.get('source', 'unknown'),
            'sourceArn': mail.get('sourceArn', 'unknown'),
            'sendingAccountId': mail.get('sendingAccountId', 'unknown')
        }
        
        # Try to get existing file
        try:
            response = s3.get_object(Bucket=BUCKET_NAME, Key=file_key)
            existing_data = json.loads(response['Body'].read().decode('utf-8'))
            if not isinstance(existing_data, list):
                existing_data = []
        except:
            existing_data = []
        
        # Append new entry
        existing_data.append(new_entry)
        
        # Save back to S3
        s3.put_object(
            Bucket=BUCKET_NAME,
            Key=file_key,
            Body=json.dumps(existing_data, indent=2),
            ContentType='application/json'
        )
        
        return {
            'statusCode': 200,
            'body': f'Successfully processed and updated {file_key}'
        }
        
    except Exception as e:
        print(f'Error processing message: {str(e)}')
        raise e</pre>
    </div>
    <p class="warning">Ensure proper indentation when copying the code!</p>
</div>

<div class="step">
    <h3>Get Role-name For Your LambdaFunction</h3>
    <div class="code-block">
        <pre>aws lambda get-function --function-name My_function --query 'Configuration.Role'</pre>
    </div>
    <p class="important">Replace role name with your actual Lambda role name!</p>
</div>

<div class="step">
    <h3>STEP 4: Configure Lambda IAM Role</h3>
    <div class="code-block">
        <pre>aws iam put-role-policy \
    --role-name My_function-role-hpm17a1r \
    --policy-name S3WritePolicy \
    --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::your-ses-notifications/*",
                "arn:aws:s3:::your-ses-notifications"
            ]
        }
    ]
}'</pre>
    </div>
    <p class="important">Replace role name with your actual Lambda role name!</p>
</div>

<div class="step">
    <h3>STEP 5: Link Lambda with SNS</h3>
    <div class="code-block">
        <pre>aws sns subscribe \
    --topic-arn arn:aws:sns:us-west-2:[Account_ID]:ses-notifications \
    --protocol lambda \
    --notification-endpoint arn:aws:lambda:us-west-2:[Account_ID]:function:My_function</pre>
    </div>
</div>

<div class="step">
    <h3>STEP 6: Add SNS Permissions to Lambda</h3>
    <div class="code-block">
        <pre>aws lambda add-permission \
    --function-name My_function \
    --statement-id SNSInvoke \
    --action lambda:InvokeFunction \
    --principal sns.amazonaws.com \
    --source-arn arn:aws:sns:us-west-2:[Account_ID]:ses-notifications</pre>
    </div>
</div>

<div class="step">
    <h3>STEP 7: Configure SES Notifications</h3>
    <div class="code-block">
        <pre>aws ses set-identity-notification-topic \
    --identity [your_domain] \
    --notification-type Bounce \
    --sns-topic arn:aws:sns:us-west-2:[Account_ID]:ses-notifications</pre>
    </div>
    <p class="tip">Repeat for Complaint and Delivery notification types.</p>
</div>

<h2>🧪 Testing the Setup</h2>

<div class="code-block">
    <pre>aws ses send-email \
    --from "contact@[your_domain]" \
    --destination "ToAddresses=success@simulator.amazonses.com" \
    --message "Subject={Data=Test Email,Charset=UTF-8},Body={Text={Data=This is a test email,Charset=UTF-8}}"</pre>
</div>

<h2>📁 Output Structure</h2>
<p>Your S3 bucket will contain:</p>
<ul>
    <li><code>your-ses-notifications/bounces/bounces-YYYY-MM.json</code></li>
    <li><code>your-ses-notifications/complaints/complaints-YYYY-MM.json</code></li>
    <li><code>your-ses-notifications/delivered/delivered-YYYY-MM.json</code></li>
</ul>

<h2>⚠️ Important Notes</h2>
<div class="warning">
    <ul>
        <li>Always use the latest AWS CLI version</li>
        <li>Replace ARNs and identifiers with your own</li>
        <li>Ensure proper IAM permissions</li>
        <li>Monitor Lambda execution limits</li>
    </ul>
</div>

<h2>🔍 Troubleshooting</h2>
<div class="tip">
    <p><strong>Common Issues:</strong></p>
    <ul>
        <li>IAM permission errors: Check role policies</li>
        <li>SNS subscription errors: Verify ARNs</li>
        <li>Lambda timeout: Adjust timeout settings</li>
        <li>S3 access denied: Check bucket policies</li>
    </ul>
</div>

</body>
</html>
