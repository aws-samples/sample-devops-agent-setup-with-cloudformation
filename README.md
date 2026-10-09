# AWS DevOps Agent - EBA CloudFormation Toolkit

> This is sample code, for non-production usage. You should work with your security and
> legal teams to meet your organizational security, regulatory and compliance requirements
> before deployment.

A set of CloudFormation templates to set up AWS DevOps Agent for a customer, step by step.
Each template does one job. Deploy only the ones the customer needs.

All templates are generic. Nothing is hardcoded. Every value (account ID, region,
names, IDs) is a parameter you pass at deploy time.

## Supported regions

AWS DevOps Agent is available in these regions (11 total, per the official AWS docs):
`us-east-1`, `us-west-2`, `ca-central-1`, `sa-east-1`, `ap-south-1`, `ap-southeast-1`, `ap-southeast-2`, `ap-northeast-1`, `eu-central-1`, `eu-west-1`, `eu-west-2`

Note: an Agent Space monitors resources across ALL regions in an associated account, so you do not need an Agent Space in every region where the customer's workloads run. Pick the Agent Space region for data residency / team proximity.
Source: https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-supported-regions.html

## Prerequisites

- AWS CLI v2.34.21 or later (must support the `aws devops-agent` command)
- Permission to create IAM roles and CloudFormation stacks
- The `--capabilities CAPABILITY_NAMED_IAM` flag on deploys that create named IAM roles
  (this is you telling CloudFormation it is allowed to create roles with names you chose)

## The templates

| Template | What it does | Deploy in |
|----------|--------------|-----------|
| T1 - Base Install | Agent Space, IAM roles, Operator App, account monitor association | Primary account |
| T2 - Webhook Add-on | SNS + Lambda + DLQ + Secrets Manager (alarm to webhook pipeline) | Primary account |
| T3 - Secondary Account Role | Cross-account investigation IAM role that trusts one Agent Space | Secondary account |
| T4 - Source Association | Links the secondary account to the Agent Space | Primary account |
| T5 - Slack Association | One-way Slack notification association | Primary account |

T1 is the only one everyone needs. The rest are optional add-ons. Deploy them in any
combination the customer wants.

## How the pieces connect

The alarm to Slack flow works like this:

```
CloudWatch Alarm (ALARM state)
        |
        v
SNS Topic
        |
        v
Lambda (signs the payload with HMAC-SHA256)
        |
        v
DevOps Agent Webhook -> Investigation -> posts to Slack
```

**IMPORTANT - we do not create the customer's alarms.** We give them the SNS topic ARN
and they add it as an alarm action on their existing alarms.

How the alarm reaches SNS: a CloudWatch alarm has an `AlarmActions` field, which is a list
of SNS topic ARNs. When the alarm goes to the ALARM state, CloudWatch automatically
publishes to every ARN in that list. So the alarm only reaches our pipeline if our SNS
topic ARN has been added to that alarm's `AlarmActions`. If it is not wired in, forcing the
alarm to ALARM does nothing. (This is also why the T2 test below works - the test alarm
must already have our SNS topic in its actions.)

---

## T1 - Base Install (deploy first)

Creates the Agent Space, the two IAM roles, the Operator App, and the account monitor
association. This is the foundation.

```bash
aws cloudformation deploy \
  --template-file T1-Base-Install.yaml \
  --stack-name devops-agent-base \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides AgentSpaceName=<AGENT_SPACE_NAME> \
  --region <REGION>
```

After it finishes, get the outputs. You will need the Agent Space ID and ARN later.

```bash
aws cloudformation describe-stacks \
  --stack-name devops-agent-base \
  --region <REGION> \
  --query 'Stacks[0].Outputs' --output table
```

T1 exports the Agent Space ID, so the add-on templates can import it automatically.

---

## T2 - Webhook Add-on (optional)

Adds the alarm pipeline. Deploy this after T1.

The webhook itself cannot be created by CloudFormation. You generate it once in the
console. Then you store the HMAC key in Secrets Manager and pass only the secret ARN to T2.
The key never goes on the command line, so it does not end up in your shell history.

Step 1 - generate the webhook in the console:
- Open the DevOps Agent console -> your Agent Space -> Capabilities -> Webhook -> Generate
- Choose HMAC as the auth type
- Copy the Webhook URL and the HMAC key (the key is shown only once)

Step 2 - store the HMAC key in Secrets Manager (pick one):

- Console (easiest): Secrets Manager -> Store a new secret -> Other type of secret ->
  Plaintext -> paste the key -> name it `<AGENT_SPACE_NAME>/webhook-hmac-key`.
- CLI: paste the key into a file with your editor (do not use `echo`), then:

```bash
aws secretsmanager create-secret \
  --name <AGENT_SPACE_NAME>/webhook-hmac-key \
  --secret-string file://hmac-key.txt \
  --region <REGION> \
  --query ARN --output text

rm hmac-key.txt
```

Paste only the key, with no extra space or new line at the end. An extra character gives a 403 from the webhook.

Copy the secret ARN. If you encrypt the secret with your own KMS key, also note the key ARN.

Step 3 - deploy T2:

```bash
aws cloudformation deploy \
  --template-file T2-Webhook-Add-on.yaml \
  --stack-name devops-agent-webhook \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    BaseStackName=devops-agent-base \
    AgentSpaceName=<AGENT_SPACE_NAME> \
    WebhookUrl=<WEBHOOK_URL> \
    HmacSecretArn=<SECRET_ARN> \
  --region <REGION>
```

Optional parameters:
- `HmacSecretKmsKeyArn` - the KMS key ARN, only if the secret uses your own KMS key.
- `DlqAlarmActionArn` - an SNS topic (for example your ops email topic) to notify when a
  failed alarm lands in the dead letter queue. Do not use the T2 alarm topic for this.

What T2 accepts:
- Only CloudWatch can publish to the topic. The topic policy denies every other publisher.
- The Lambda only forwards CloudWatch alarm JSON from this account. Other messages are
  dropped and logged (without the message body).
- If the webhook call fails, the Lambda raises the error. Lambda retries 2 times, then puts
  the event in the dead letter queue, and the `<AGENT_SPACE_NAME>-trigger-dlq-not-empty`
  alarm goes to ALARM.

Keep `AgentSpaceName` the same as in T1. T2 names its resources from it.

Test it - point an alarm at the SNS topic and force it into ALARM:

```bash
aws cloudwatch set-alarm-state \
  --alarm-name <YOUR_ALARM_NAME> \
  --state-value ALARM \
  --state-reason "Test" \
  --region <REGION>

aws logs tail /aws/lambda/<AGENT_SPACE_NAME>-alarm-trigger --region <REGION> --since 3m
```

Look for `Sent: 200` in the log. That means the webhook accepted the request.

### Good to know about T2 (encryption)

T2 creates one KMS key. It encrypts the SNS topic and the dead letter queue.
The HMAC secret is created by you (step 2), so you choose its encryption key.
- Cost: a KMS key costs about 1 USD per month, plus a very small charge per request.
- Delete: when you delete the T2 stack, AWS does not remove the key at once. It waits up to 30 days before the key is fully deleted. This is normal KMS behaviour.

---

## T3 and T4 - Cross-account monitoring (optional)

This lets the Agent Space investigate incidents in a second AWS account.
T3 goes in the secondary account. T4 goes in the primary account.

### Which account should be the primary account?

Do not use your AWS Organizations management account as the primary (monitoring) account,
and do not deploy from it with `OrganizationAccountAccessRole`. That role has full admin
rights in every member account. Best practice is to keep workloads and tools out of the
management account.

Use a dedicated monitoring (or shared tooling) account for the Agent Space. To deploy T3 in
each secondary account, use a scoped deployment role that can only manage CloudFormation
and the IAM role T3 creates. For many accounts, CloudFormation StackSets with
service-managed permissions is a good option.

See [Management account in the AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/management-account.html).

### Switch to the secondary account (CLI)

This example assumes a scoped deployment role (`<DEPLOYMENT_ROLE_NAME>`) that already
exists in the secondary account and trusts your deployment identity.

```bash
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::<SECONDARY_ACCOUNT_ID>:role/<DEPLOYMENT_ROLE_NAME> \
  --role-session-name devops-agent-setup \
  --output json)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | python3 -c "import sys,json; print(json.load(sys.stdin)['Credentials']['AccessKeyId'])")
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | python3 -c "import sys,json; print(json.load(sys.stdin)['Credentials']['SecretAccessKey'])")
export AWS_SESSION_TOKEN=$(echo $CREDS | python3 -c "import sys,json; print(json.load(sys.stdin)['Credentials']['SessionToken'])")

aws sts get-caller-identity
```

Confirm the account ID shows the secondary account. The session lasts one hour. If it
expires, run this block again.

### T3 - deploy in the secondary account

```bash
aws cloudformation deploy \
  --template-file T3-Secondary-Account-Role.yaml \
  --stack-name devops-agent-secondary \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    MonitoringAccountId=<PRIMARY_ACCOUNT_ID> \
    AgentSpaceArn=<AGENT_SPACE_ARN_FROM_T1> \
  --region <REGION>
```

If an IAM role with the same name already exists in this account (for example from an
earlier test), the deploy fails with `ResourceExistenceCheck failed`. Delete the old role,
or pass a new name with `RoleName=<NEW_ROLE_NAME>`.

`AgentSpaceArn` is the `AgentSpaceArn` output of the T1 stack. The role trusts only that
one Agent Space. Add `DenySensitiveReads=Yes` if you want the optional Deny (see Security).

Get the role ARN from the outputs. You need it for T4.

```bash
aws cloudformation describe-stacks \
  --stack-name devops-agent-secondary \
  --region <REGION> \
  --query 'Stacks[0].Outputs' --output table
```

### Switch back to the primary account

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
aws sts get-caller-identity
```

### T4 - deploy in the primary account

```bash
aws cloudformation deploy \
  --template-file T4-Source-Association.yaml \
  --stack-name devops-agent-source-assoc \
  --parameter-overrides \
    BaseStackName=devops-agent-base \
    ServiceAccountId=<SECONDARY_ACCOUNT_ID> \
    ServiceAccountRoleArn=<ROLE_ARN_FROM_T3> \
  --region <REGION>
```

T4 imports the Agent Space ID from the T1 stack automatically.

---

## T5 - Slack (optional)

Slack has two parts. This is the most confusing bit, so read carefully.

### What is console-only and what is CFN

- The Slack registration (the OAuth login where you pick your workspace and click Allow)
  is browser-only. No template can do this. It is a one-time step.
- After registration, the channel association can be done by T5 (as code) or in the console.
- Bidirectional mode (where you @mention the agent and it replies in the thread) is
  console-only. CloudFormation does not support it. T5 does one-way notifications only.

So for a customer:
1. Console: register Slack once (OAuth). This creates a Slack ServiceId.
2. T5: deploy the one-way association as code.
3. Console (optional): if they want two-way chat, set up bidirectional. It needs a
   PRIVATE channel.

### Register Slack (console, one time)

- Console -> your Agent Space -> Communications -> Add integration -> Slack -> Register
- Pick the workspace (must NOT be Enterprise Grid) and click Allow

### Get the values T5 needs

```bash
aws devops-agent list-services --region <REGION>
```

This gives you the Slack `serviceId` and the workspace `teamId`. For the channel ID,
right-click the channel in Slack -> View channel details -> scroll to the bottom.

### Deploy T5

```bash
aws cloudformation deploy \
  --template-file T5-Slack-Association.yaml \
  --stack-name devops-agent-slack \
  --parameter-overrides \
    BaseStackName=devops-agent-base \
    SlackServiceId=<SLACK_SERVICE_ID> \
    SlackWorkspaceId=<WORKSPACE_ID> \
    SlackWorkspaceName=<WORKSPACE_NAME> \
    SlackChannelId=<CHANNEL_ID> \
    SlackChannelName=<CHANNEL_NAME> \
  --region <REGION>
```

Last step - invite the app to the channel in Slack, or it can not post:

```
/invite @AWS DevOps Agent - <REGION LABEL>
```

### If you want bidirectional (two-way chat)

- Use a PRIVATE channel. Bidirectional is not supported on public channels.
- Set it up in the console association step with Bidirectional mode enabled.
- The console auto-creates a channel IAM role for it.

---

## Common errors and fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `aws devops-agent` not found | CLI too old | Upgrade to v2.34.21 or later |
| Early validation: ResourceExistenceCheck failed | A named resource (usually an IAM role) already exists from an earlier run | Use a different name via the RoleName/AgentSpaceName parameter, or delete the old resource first |
| Role will not delete ("must delete policies first") | The role has an inline or attached policy | Delete the inline policy and detach managed policies, then delete the role |
| ExpiredToken | The assume-role session (1 hour) expired | Run the assume-role block again |
| Failed to associate Slack - only supported in private channels | Bidirectional turned on for a public channel | Use a private channel, or turn bidirectional off |
| Lambda returns non-200 | Wrong HMAC key | Regenerate the webhook, update the secret value in Secrets Manager, then wait a few minutes or update the Lambda so it reloads the key |
| `Dropped SNS message ...` in the Lambda log | The message was not CloudWatch alarm JSON from this account | Expected. Only CloudWatch alarms are forwarded |

---

## Cleanup

Delete stacks in reverse order (add-ons first, base last). For the secondary account
stack, switch into that account first.

```bash
aws cloudformation delete-stack --stack-name <STACK_NAME> --region <REGION>
aws cloudformation wait stack-delete-complete --stack-name <STACK_NAME> --region <REGION>
```


## Security

What the agent can do:
T1 and T3 attach the AWS managed policy `AIDevOpsAgentAccessPolicy` to the agent role.
It is mostly read access, but it is not read-only. In every account you connect, it also
allows the agent to:
- create AWS Support cases (`support:CreateCase`) and run CloudTrail Lake queries
- read SSM parameter values (`ssm:GetParameters`) and SSM documents
- read Lambda configuration, including environment variables (`lambda:GetFunctionConfiguration`)
- read CloudWatch Logs (`logs:GetLogEvents`, `logs:FilterLogEvents`, `logs:StartQuery`)
- use the Kubernetes API of EKS clusters (`eks:AccessKubernetesApi`)

Check the current policy before you deploy, because AWS updates it:
[AIDevOpsAgentAccessPolicy](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AIDevOpsAgentAccessPolicy.html).
If these accounts hold secrets in Lambda environment variables, SSM parameters or logs, move
them to Secrets Manager first.

Creating support cases is a feature of DevOps Agent, so the templates keep it. If you do not
want the agent to open cases, add a Deny for `support:CreateCase` yourself.

Optional Deny: deploy T1 and T3 with `DenySensitiveReads=Yes`. This adds a Deny policy to the
agent role for `ssm:GetParameter*`, `lambda:GetFunctionConfiguration`
and `appsync:GetGraphqlApiEnvironmentVariables`. The agent can then give less detail in some
investigations. Logs are not denied, because the agent needs them. Add more actions if your
data needs it.

Other controls in the templates:
- Every agent role trusts only `aidevops.amazonaws.com`, with `aws:SourceAccount`. The T3
  role trusts only the one Agent Space ARN you pass in.
- You store the webhook HMAC key in Secrets Manager. The template and the Lambda only get its ARN.
- Only CloudWatch can publish to the T2 SNS topic, and the Lambda only forwards CloudWatch
  alarm JSON from its own account.
- The Lambda only sends a signed request to the DevOps Agent webhook URL you provide.

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
