
Trigger Jenkins builds from Slack commands automatically  
(no plugins, no extra backend — just n8n)  

⭐ Highlights
✔ Converts Slack build commands to Jenkins jobs  
✔ Easily customize environments & branches  
✔ Works self-hosted with just n8n





# n8n-Slack-Jenkins-Automation

1. Workflow Node Overview

Your n8n workflow includes exactly these nodes:

Slack Trigger → receives Slack messages/events

Parse Slack Message (Function node) → extracts service, branchName, environment

Service Switch (Switch node) → routes based on service value

Trigger Jenkins - <service> (HTTP Request) → calls Jenkins buildWithParameters

Do not rename the JSON fields from the Function node.
Downstream nodes expect:

service
branchName
environment





2. Creating the Slack App (EXACT UI STEPS)
Step 1 — Create App

Visit → https://api.slack.com/apps

Click Create New App → From scratch

App Name: jenkins-deploy-bot

Workspace: choose your workspace

Click Create App




Step 2 — Add Bot Token Scopes

Left sidebar → OAuth & Permissions → Scopes → Bot Token Scopes
Add these EXACT scopes:

app_mentions:read

channels:history

channels:read

chat:write

groups:history

im:history

Scroll to top → click Install to Workspace → approve.

Copy Bot User OAuth Token (xoxb-...).
Save it — required for n8n.





Step 3 — Signing Secret

Left sidebar → Basic Information → App Credentials → Signing Secret
Copy it (optional but recommended).






Step 4 — App Home & Invite Bot

Left sidebar → App Home → set Display Name:

jenkins-deploy-bot


Then in Slack:

/invite @jenkins-deploy-bot





3. Event Subscriptions (EXACT STEPS)

Slack must validate an n8n webhook URL with a challenge.

You must get the webhook URL from n8n AFTER importing the workflow (Part 4).

Steps:

Left sidebar → Event Subscriptions

Switch ON

Request URL → paste the Production Webhook URL from Slack Trigger node
Example format:

http://<PUBLIC_IP>:5678/<parameters from the n8n url as it is >


Under Subscribe to bot events → click Add Bot User Event
Add:

app_mention


(Optional: add message.channels if you want all messages)

Save changes

Slack will send a challenge. It succeeds only if:

workflow is active

Slack Trigger node has credentials

URL is public and reachable




4. Importing the workflow into n8n (EXACT STEPS)

Open n8n in browser:

http://<YOUR_PUBLIC_IP>:5678


Go → Workflows → Import from File → choose slack-jenkins Automation.json

The workflow loads

DO NOT activate yet

Continue with credential setup below





5. Adding Credentials in n8n (NODE-BY-NODE EXACT guide)
Node 1 — Slack Trigger

Open the Slack Trigger node

Click Credentials → Create New

Choose credential type → Slack API

Enter:

Access Token → paste your Bot User OAuth Token (xoxb-)

Signing Secret → paste Slack Signing Secret

Name → Slack - jenkins-deploy-bot

Save

Select this credential in the Slack Trigger node

The node now shows a Production Webhook URL

Copy that URL → paste into Slack Event Subscriptions Request URL (Section 3)

Node 2 — Parse Slack Message (Function)

No credentials needed.
Paste EXACTLY this code:

// Extract Slack text
let rawText = $json.text ? $json.text.trim() : "";

// Remove leading Slack mentions like <@U0A1EBWG5M3>
rawText = rawText.replace(/<@[^>]+>/g, "").trim();

const text = rawText;

// SERVICE → word after "build"
let svcMatch = text.match(/build\s+([A-Za-z0-9_\-]+)/i);
let service = svcMatch ? svcMatch[1] : null;

// BRANCH → anything after service and before "in"
let branchMatch = text.match(/build\s+[A-Za-z0-9_\-]+\s+([A-Za-z0-9_\-]+)\s+in/i);
let branchName = branchMatch ? branchMatch[1] : null;

// ENVIRONMENT
let envMatch = text.match(/\b(dev|development|stage|staging|mock|uat|prod|production)\b/i);
let environment = envMatch ? envMatch[1].toLowerCase() : null;

const envMap = {
  development: "dev",
  dev: "dev",
  stage: "stage",
  staging: "stage",
  mock: "mock",
  uat: "uat",
  production: "prod",
  prod: "prod"
};

if (environment && envMap[environment]) environment = envMap[environment];

return {
  json: {
    rawMessage: rawText,
    service,
    branchName,
    environment
  }
};





Node 3 — Service Switch

Open Service Switch

Set Mode = Rules

Add rules EXACTLY like this:

Example

Rule 1:



Value 1 → expression → {{$json.service}}

Condition → is equal to

Value 2 → crudops



Rule 2:

{{$json.service}} is equal to sales

Rule 3:

{{$json.service}} is equal to dms_api_gateway

Final Rule:

{{$json.service}} is set

n8n will create output ports for each rule.
Connect each port to its corresponding Jenkins trigger node.



Node 4 — HTTP Request: Trigger Jenkins - <service>

You will have one such node per service.

Open the node and set:

HTTP Settings

Method: POST

URL:

http://<JENKINS_IP>:8080/job/<JOB_NAME>/buildWithParameters


Example:

http://34.228.69.35:8080/job/crudops/buildWithParameters

Authentication

Credentials → Create New → HTTP Basic Auth

Username: your Jenkins username (Example: Parimal)

Password: Jenkins API Token (Example: 116e7f9cee6a...)

Save as → Jenkins - Parimal

Select this credential for the node.

Query Parameters (IMPORTANT — must match Jenkins parameters)

Add these two parameters:

Name	Value Expression
Environment	{{$json.environment}}
BranchName	{{$json.branchName}}

Do not put quotes around expressions.

Headers

Leave empty unless Jenkins needs crumb.
If you get 403 CSRF, follow next section.




6. Jenkins Crumb Setup (ONLY if 403 error)

If Jenkins returns:

403 No valid crumb


Do this:

Node — Get Jenkins Crumb

Add new HTTP Request node

Method: GET

URL:

http://<JENKINS_IP>:8080/crumbIssuer/api/json


Auth → use Jenkins - Parimal Basic Auth credential

Then modify the Trigger Jenkins node:

Add header:

Header Name	Header Value Expression
Jenkins-Crumb	{{ $node["Get Jenkins Crumb"].json.crumb }}

Connect Crumb → Trigger Jenkins.




7. Activating & Testing (EXACT STEPS)

Click Activate workflow in n8n

In Slack channel, make sure bot is added:

/invite @jenkins-deploy-bot


Send EXACT message:

@jenkins-deploy-bot build crudops ec2-deploy-branch in stage


Go to n8n → Executions

Check nodes:

Slack Trigger

Shows event from Slack.

Parse Slack Message

Expected output:

{
  "rawMessage": "build crudops ec2-deploy-branch in stage",
  "service": "crudops",
  "branchName": "ec2-deploy-branch",
  "environment": "stage"
}

Service Switch

Shows which rule matched.

Trigger Jenkins

Response should be 200/201/302.
Jenkins should start a build.




8. Checklist after Import (DO THESE IN THIS ORDER)

Open workflow

Slack Trigger → attach Slack credential

Copy webhook URL → paste into Slack Event Subscriptions

Parse Slack Message → ensure code is correct

Service Switch → ensure rules use {{$json.service}}

Trigger Jenkins nodes → set correct job URLs

Create Jenkins Basic Auth credential → attach

Ensure query params match Jenkins:

Environment

BranchName

Activate workflow

Test from Slack




9. Requirements You Must Provide

Slack Bot Token

Slack Signing Secret

Jenkins IP

Jenkins username

Jenkins API Token

Exact Jenkins job names
