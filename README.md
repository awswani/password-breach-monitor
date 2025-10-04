# Password Breach Monitor

Automated system to monitor email addresses for data breaches using the HaveIBeenPwned API. Sends instant alerts when your credentials appear in new breaches.

![AWS](https://img.shields.io/badge/AWS-Lambda-orange?logo=amazon-aws)
![EventBridge](https://img.shields.io/badge/AWS-EventBridge-orange?logo=amazon-aws)
![DynamoDB](https://img.shields.io/badge/AWS-DynamoDB-orange?logo=amazon-aws)
![Security](https://img.shields.io/badge/Security-Monitoring-red)

## 🎯 Overview

This serverless application automatically checks if your email addresses have been compromised in data breaches by:
- Running daily automated checks via EventBridge
- Querying the HaveIBeenPwned API
- Storing breach history in DynamoDB to prevent duplicate alerts
- Sending instant email notifications for new breaches
- Providing actionable security recommendations

**Built for**: Portfolio demonstration | Personal security monitoring | Enterprise security teams

## 🏗️ Architecture

```
┌──────────────────┐
│  EventBridge     │
│  (Daily 9am)     │
└────────┬─────────┘
         │ Trigger
         ▼
┌──────────────────┐
│  Lambda Function │
│  breach-checker  │
└────────┬─────────┘
         │
         ├──────────────────────┐
         │                      │
         ▼                      ▼
┌──────────────────┐   ┌──────────────────┐
│ HaveIBeenPwned   │   │    DynamoDB      │
│      API         │   │ (Breach History) │
└──────────────────┘   └──────────────────┘
         │                      │
         └──────────┬───────────┘
                    │
                    ▼
           ┌──────────────────┐
           │   SNS Topic      │
           │  (Email Alerts)  │
           └────────┬─────────┘
                    │
                    ▼
           ┌──────────────────┐
           │  Your Email      │
           └──────────────────┘
```

## ✨ Features

- **Automated Daily Checks** - EventBridge triggers Lambda every day at 9 AM
- **Real-Time Alerts** - Instant email notifications for new breaches
- **Smart Deduplication** - DynamoDB tracks reported breaches to prevent alert spam
- **Detailed Breach Info** - Includes breach date, compromised data types, and affected count
- **Actionable Guidance** - Provides immediate security recommendations
- **Multi-Email Support** - Monitor multiple email addresses simultaneously
- **Cost-Effective** - Runs entirely on AWS free tier
- **Serverless** - No infrastructure to manage
- **Secure** - API keys stored in environment variables

## 🛠️ Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Compute** | AWS Lambda (Node.js 18) | Breach checking logic |
| **Scheduling** | Amazon EventBridge | Daily automated triggers |
| **Database** | Amazon DynamoDB | Breach history tracking |
| **Notifications** | Amazon SNS | Email alerts |
| **API** | HaveIBeenPwned v3 | Breach data source |

## 📦 Setup Instructions

### Prerequisites

- AWS Account
- HaveIBeenPwned API Key (free from https://haveibeenpwned.com/API/Key)
- AWS CLI configured
- Email address for alerts

### Step 1: Get HaveIBeenPwned API Key

1. Visit https://haveibeenpwned.com/API/Key
2. Purchase API key ($3.50/month - supports the service)
3. Save the API key securely

### Step 2: Create DynamoDB Table

```bash
aws dynamodb create-table \
    --table-name password-breach-monitor \
    --attribute-definitions \
        AttributeName=email,AttributeType=S \
        AttributeName=breachName,AttributeType=S \
    --key-schema \
        AttributeName=email,KeyType=HASH \
        AttributeName=breachName,KeyType=RANGE \
    --billing-mode PAY_PER_REQUEST \
    --region us-east-1
```

### Step 3: Create SNS Topic

```bash
# Create topic
aws sns create-topic --name password-breach-alerts

# Subscribe your email
aws sns subscribe \
    --topic-arn arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:password-breach-alerts \
    --protocol email \
    --notification-endpoint your-email@example.com

# Confirm subscription in your email inbox
```

### Step 4: Create Lambda Function

1. **Create IAM Role for Lambda:**

```bash
# Create trust policy
cat > lambda-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create role
aws iam create-role \
    --role-name PasswordBreachMonitorRole \
    --assume-role-policy-document file://lambda-trust-policy.json

# Attach policies
aws iam attach-role-policy \
    --role-name PasswordBreachMonitorRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Create custom policy for DynamoDB and SNS
cat > lambda-permissions.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:YOUR_ACCOUNT_ID:table/password-breach-monitor"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:password-breach-alerts"
    }
  ]
}
EOF

aws iam put-role-policy \
    --role-name PasswordBreachMonitorRole \
    --policy-name BreachMonitorPermissions \
    --policy-document file://lambda-permissions.json
```

2. **Create Lambda Function:**

```bash
# Package Lambda code
zip -j breach-checker.zip lambda/breach-checker.mjs

# Create function
aws lambda create-function \
    --function-name password-breach-checker \
    --runtime nodejs18.x \
    --role arn:aws:iam::YOUR_ACCOUNT_ID:role/PasswordBreachMonitorRole \
    --handler breach-checker.handler \
    --zip-file fileb://breach-checker.zip \
    --timeout 60 \
    --memory-size 256 \
    --environment Variables="{
        HIBP_API_KEY=your-api-key-here,
        MONITORED_EMAILS=email1@example.com,email2@example.com,
        SNS_TOPIC_ARN=arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:password-breach-alerts,
        TABLE_NAME=password-breach-monitor
    }"
```

### Step 5: Set Up EventBridge Schedule

```bash
# Create rule (runs daily at 9 AM UTC)
aws events put-rule \
    --name password-breach-daily-check \
    --schedule-expression "cron(0 9 * * ? *)" \
    --state ENABLED

# Add Lambda as target
aws events put-targets \
    --rule password-breach-daily-check \
    --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:YOUR_ACCOUNT_ID:function:password-breach-checker"

# Grant EventBridge permission to invoke Lambda
aws lambda add-permission \
    --function-name password-breach-checker \
    --statement-id AllowEventBridgeInvoke \
    --action lambda:InvokeFunction \
    --principal events.amazonaws.com \
    --source-arn arn:aws:events:us-east-1:YOUR_ACCOUNT_ID:rule/password-breach-daily-check
```

### Step 6: Test the Function

```bash
# Manual test
aws lambda invoke \
    --function-name password-breach-checker \
    --payload '{}' \
    response.json

# Check the output
cat response.json
```

## 📧 Sample Alert

When a breach is detected, you'll receive an email like this:

```
Subject: 🚨 BREACH ALERT: your@email.com - Adobe

🚨 PASSWORD BREACH ALERT 🚨

Email Address: your@email.com
Breach: Adobe
Breach Date: 2013-10-04
Compromised Accounts: 152,445,165

Description:
In October 2013, 153 million Adobe accounts were breached with each
containing an internal ID, username, email, encrypted password and a
password hint in plain text.

Compromised Data:
Email addresses, Password hints, Passwords, Usernames

⚠️ IMMEDIATE ACTIONS REQUIRED:
1. Change your password IMMEDIATELY if you still use it
2. Enable two-factor authentication (2FA)
3. Check for unauthorized account activity
4. Update passwords on any sites using the same credentials
5. Consider using a password manager

More Info: https://haveibeenpwned.com/

---
This alert was generated by your Password Breach Monitor
Detected at: 2025-10-04T09:00:23.456Z
```

## 🔧 Configuration

### Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `HIBP_API_KEY` | HaveIBeenPwned API key | `abc123...` |
| `MONITORED_EMAILS` | Comma-separated email list | `email1@test.com,email2@test.com` |
| `SNS_TOPIC_ARN` | SNS topic for alerts | `arn:aws:sns:...` |
| `TABLE_NAME` | DynamoDB table name | `password-breach-monitor` |

### Customizing Schedule

Change the EventBridge cron expression:

| Schedule | Cron Expression |
|----------|----------------|
| Daily at 9 AM | `cron(0 9 * * ? *)` |
| Every 6 hours | `rate(6 hours)` |
| Weekly Monday 9 AM | `cron(0 9 ? * MON *)` |

## 💰 Cost Analysis

**Monthly Cost: ~$0.10** (within free tier limits)

| Service | Usage | Free Tier | Cost |
|---------|-------|-----------|------|
| Lambda | 30 invocations/month | 1M free | $0.00 |
| DynamoDB | 100 writes/month | 25GB free | $0.00 |
| SNS | 5 emails/month | 1,000 free | $0.00 |
| EventBridge | 30 triggers/month | 14M free | $0.00 |
| **HIBP API** | | | **$3.50/month** |
| **Total** | | | **$3.50/month** |

*The only real cost is the HaveIBeenPwned API subscription*

## 🛡️ Security Best Practices

1. **API Key Security**
   - Store API key in Lambda environment variables
   - Never commit API keys to Git
   - Rotate keys periodically

2. **Least Privilege IAM**
   - Lambda only has DynamoDB/SNS permissions
   - No unnecessary access granted

3. **Email Privacy**
   - Emails stored only in environment variables
   - Not logged to CloudWatch (sensitive)

4. **Breach Response**
   - Act on alerts within 24 hours
   - Change passwords immediately
   - Enable 2FA everywhere

## 📊 Monitoring

### View Lambda Logs

```bash
aws logs tail /aws/lambda/password-breach-checker --follow
```

### Check DynamoDB Entries

```bash
aws dynamodb scan --table-name password-breach-monitor
```

### Test SNS Notifications

```bash
aws sns publish \
    --topic-arn arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:password-breach-alerts \
    --message "Test breach alert" \
    --subject "Test Alert"
```

## 🚀 Future Enhancements

- [ ] Slack/Teams integration
- [ ] Web dashboard with breach history
- [ ] Password strength checking
- [ ] Integration with 1Password/LastPass
- [ ] Dark web monitoring
- [ ] Multi-region deployment
- [ ] Custom breach severity scoring
- [ ] Automated password rotation

## 🐛 Troubleshooting

### No Emails Received

1. **Check SNS subscription**: `aws sns list-subscriptions`
2. **Verify email confirmed**: Check spam folder for confirmation
3. **Test SNS manually**: See monitoring section

### Lambda Timeout

- Increase timeout: `aws lambda update-function-configuration --function-name password-breach-checker --timeout 120`

### API Rate Limiting

- HaveIBeenPwned allows 10 requests per minute
- Add delays if monitoring many emails

### DynamoDB Access Denied

- Verify IAM permissions include `dynamodb:PutItem` and `dynamodb:Query`

## 📚 API Reference

**HaveIBeenPwned API v3**
- Documentation: https://haveibeenpwned.com/API/v3
- Rate Limit: 10 requests/minute
- Response Format: JSON array of breaches

**Breach Object:**
```json
{
  "Name": "Adobe",
  "Title": "Adobe",
  "Domain": "adobe.com",
  "BreachDate": "2013-10-04",
  "AddedDate": "2013-12-04T00:00Z",
  "ModifiedDate": "2022-05-15T23:52Z",
  "PwnCount": 152445165,
  "Description": "...",
  "DataClasses": ["Email addresses", "Passwords", ...]
}
```

## 🤝 Contributing

This is a portfolio project, but improvements are welcome!

Potential contributions:
- Additional notification channels
- Enhanced breach analysis
- UI/dashboard development
- Testing framework

## 📄 License

MIT License - Use freely for personal or commercial projects

## 👤 Author

**Wani Lado**
- Portfolio: [cloudwani.tech](https://cloudwani.tech)
- LinkedIn: [linkedin.com/in/wani-lado615](https://linkedin.com/in/wani-lado615)
- GitHub: [@awswani](https://github.com/awswani)
- Email: wani.lado615@gmail.com

---

⭐ Stay secure! If this project helps protect your accounts, please star it!

**Disclaimer**: This tool monitors publicly available breach data. It cannot detect all security incidents. Always practice good password hygiene and enable 2FA.
