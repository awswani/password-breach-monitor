# Implementation Guide - Step by Step

This guide walks you through deploying the Password Breach Monitor from scratch.

## 🚀 Quick Start (30 minutes)

### Phase 1: Get API Key (5 min)

1. Go to https://haveibeenpwned.com/API/Key
2. Purchase API key ($3.50/month)
3. Save the key - you'll need it soon

### Phase 2: AWS Setup (10 min)

**1. Create DynamoDB Table**

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

**2. Create SNS Topic**

```bash
# Create topic
TOPIC_ARN=$(aws sns create-topic --name password-breach-alerts --query 'TopicArn' --output text)
echo "Topic ARN: $TOPIC_ARN"

# Subscribe your email (replace with your email)
aws sns subscribe \
    --topic-arn $TOPIC_ARN \
    --protocol email \
    --notification-endpoint wani.lado615@gmail.com

# Check your email and CLICK THE CONFIRMATION LINK!
```

### Phase 3: Deploy Lambda (10 min)

**1. Create IAM Role**

```bash
# Create trust policy file
cat > /tmp/lambda-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create role
ROLE_ARN=$(aws iam create-role \
    --role-name PasswordBreachMonitorRole \
    --assume-role-policy-document file:///tmp/lambda-trust.json \
    --query 'Role.Arn' --output text)
echo "Role ARN: $ROLE_ARN"

# Attach basic execution policy
aws iam attach-role-policy \
    --role-name PasswordBreachMonitorRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Get your account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create permissions policy
cat > /tmp/lambda-perms.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem", "dynamodb:Query"],
      "Resource": "arn:aws:dynamodb:us-east-1:${ACCOUNT_ID}:table/password-breach-monitor"
    },
    {
      "Effect": "Allow",
      "Action": ["sns:Publish"],
      "Resource": "${TOPIC_ARN}"
    }
  ]
}
EOF

aws iam put-role-policy \
    --role-name PasswordBreachMonitorRole \
    --policy-name BreachMonitorPermissions \
    --policy-document file:///tmp/lambda-perms.json

# Wait for role to propagate
echo "Waiting 10 seconds for IAM role to propagate..."
sleep 10
```

**2. Package and Deploy Lambda**

```bash
# Package Lambda code
cd /Users/wani/password-breach-monitor
zip -j breach-checker.zip lambda/breach-checker.mjs

# Deploy (replace placeholders!)
aws lambda create-function \
    --function-name password-breach-checker \
    --runtime nodejs18.x \
    --role $ROLE_ARN \
    --handler breach-checker.handler \
    --zip-file fileb://breach-checker.zip \
    --timeout 60 \
    --memory-size 256 \
    --environment Variables="{
        HIBP_API_KEY=YOUR_API_KEY_HERE,
        MONITORED_EMAILS=email1@example.com\\,email2@example.com,
        SNS_TOPIC_ARN=${TOPIC_ARN},
        TABLE_NAME=password-breach-monitor
    }"

# Note: Update the HIBP_API_KEY and MONITORED_EMAILS before running!
```

### Phase 4: Schedule It (5 min)

```bash
# Get Lambda ARN
FUNCTION_ARN=$(aws lambda get-function --function-name password-breach-checker --query 'Configuration.FunctionArn' --output text)

# Create EventBridge rule (daily at 9 AM UTC = 4 AM EST / 1 AM PST)
RULE_ARN=$(aws events put-rule \
    --name password-breach-daily-check \
    --schedule-expression "cron(0 9 * * ? *)" \
    --state ENABLED \
    --query 'RuleArn' --output text)

# Add Lambda as target
aws events put-targets \
    --rule password-breach-daily-check \
    --targets "Id"="1","Arn"="$FUNCTION_ARN"

# Give EventBridge permission to invoke Lambda
aws lambda add-permission \
    --function-name password-breach-checker \
    --statement-id AllowEventBridgeInvoke \
    --action lambda:InvokeFunction \
    --principal events.amazonaws.com \
    --source-arn $RULE_ARN
```

### Phase 5: Test It! (5 min)

```bash
# Test the Lambda function manually
aws lambda invoke \
    --function-name password-breach-checker \
    --payload '{}' \
    /tmp/response.json

# Check the response
cat /tmp/response.json

# Check CloudWatch Logs
aws logs tail /aws/lambda/password-breach-checker --follow
```

## ✅ Verification Checklist

- [ ] DynamoDB table created and shows "ACTIVE"
- [ ] SNS topic created
- [ ] Email subscription **CONFIRMED** (check spam!)
- [ ] Lambda function deployed successfully
- [ ] EventBridge rule is "ENABLED"
- [ ] Test invocation succeeded
- [ ] Received test email (if breach found)

## 🎯 Expected Results

**First Run:**
- If monitored emails have breaches → You'll get email alerts
- Lambda logs show: "Found X total breaches"
- DynamoDB will have entries for each breach

**Subsequent Runs:**
- Only NEW breaches trigger alerts
- Logs show: "Breach already reported"
- No duplicate emails

## 🔧 Updating Configuration

**Add More Emails:**
```bash
aws lambda update-function-configuration \
    --function-name password-breach-checker \
    --environment Variables="{
        HIBP_API_KEY=your-key,
        MONITORED_EMAILS=email1@test.com\\,email2@test.com\\,email3@test.com,
        SNS_TOPIC_ARN=$TOPIC_ARN,
        TABLE_NAME=password-breach-monitor
    }"
```

**Change Schedule:**
```bash
# Run every 6 hours instead of daily
aws events put-rule \
    --name password-breach-daily-check \
    --schedule-expression "rate(6 hours)" \
    --state ENABLED
```

## 🧹 Cleanup (if you want to delete everything)

```bash
# Delete EventBridge rule
aws events remove-targets --rule password-breach-daily-check --ids "1"
aws events delete-rule --name password-breach-daily-check

# Delete Lambda
aws lambda delete-function --function-name password-breach-checker

# Delete IAM role
aws iam delete-role-policy --role-name PasswordBreachMonitorRole --policy-name BreachMonitorPermissions
aws iam detach-role-policy --role-name PasswordBreachMonitorRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name PasswordBreachMonitorRole

# Delete SNS topic
aws sns delete-topic --topic-arn $TOPIC_ARN

# Delete DynamoDB table
aws dynamodb delete-table --table-name password-breach-monitor
```

## 💡 Pro Tips

1. **Test with a known breached email** - Use a disposable email or old account to verify alerts work
2. **Check spam folder** - SNS confirmation emails often go to spam
3. **Monitor costs** - Set up billing alerts for peace of mind
4. **API key security** - Never commit it to Git!
5. **Response time** - Act on alerts within 24 hours

## 🚨 Common Issues

**"Email not confirmed"**
- Check spam/junk folder for SNS confirmation
- Resend: `aws sns subscribe ...` (see Phase 2)

**"Access Denied" on DynamoDB**
- IAM role needs more time to propagate (wait 30 sec)
- Verify policy is attached: `aws iam get-role-policy --role-name PasswordBreachMonitorRole --policy-name BreachMonitorPermissions`

**"No breaches found" but you know there are breaches**
- Verify HIBP API key is correct
- Check API key hasn't expired
- Ensure email is correctly formatted

**Lambda timeout**
- Increase timeout: `aws lambda update-function-configuration --function-name password-breach-checker --timeout 120`

Ready to deploy? Start with **Phase 1**! 🚀
