# Application Logging Configuration Lab

## Overview

This project deploys a small Flask application on an EC2 instance (`logging-lab`,
t3.micro, Amazon Linux 2023) that emits structured JSON logs via `structlog`. The
CloudWatch Logs Agent tails the application's log file and ships each line to
CloudWatch Logs, where it can be searched and aggregated with Logs Insights.

- **App:** 
`app/server.py` — Flask app with `/`, `/health`, `/order` (POST), and
  `/error` routes, each logging a structured event (`request_received`,
  `health_check`, `order_created`, `request_failed`) with a `correlation_id`.
- **Log group:** `/aws/application/api`
- **Log stream:** named after the EC2 instance ID (`{instance_id}`)
- **Retention:** 30 days

## Deployment Steps

1. **Application** 
— Wrote `app/server.py` with structlog configured to render
   JSON lines to `application.log`. Dependencies (`flask`, `structlog`, `boto3`)
   listed in `app/requirements.txt`.

2. **CloudWatch Agent install** 
— On the EC2 instance, downloaded and installed
   the agent via `wget` + `sudo rpm -U`.

3. **IAM role** 
— Created `CloudWatchAgentRole` with a trust policy allowing
   `ec2.amazonaws.com` to assume it, attached a custom `CloudWatchLogsPolicy`
   (`CreateLogGroup`, `CreateLogStream`, `PutLogEvents`, `DescribeLogStreams`),
   wrapped it in an instance profile (`CloudWatchAgentProfile`), and associated
   that profile with the instance. No access keys stored on the box.

4. **Agent configuration** 
— Saved `config/cloudwatch-agent-config.json` to
   `/opt/aws/amazon-cloudwatch-agent/etc/config.json` on the instance, pointing
   at `/home/ec2-user/app/application.log`, then started the agent with
   `amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s`.

5. **Deploy and run** 
— Copied `app/` to the instance via `scp`, installed
   `python3-pip` and dependencies with `pip3 install -r requirements.txt`, and
   ran `python3 server.py`.

6. **Generate traffic** 
— Hit `/`, `/health`, and `/order` with `curl`, plus a
   loop of 50 concurrent requests to `/`, to produce a realistic volume of logs.

7. **Retention policy** 
— Set to 30 days via
   `aws logs put-retention-policy --log-group-name /aws/application/api --retention-in-days 30`
   (see `config/retention-policy.txt` for reasoning).

## How to Verify Logs Are Working

From a local terminal with AWS CLI configured:

```bash
aws logs describe-log-groups --log-group-name-prefix /aws/application/api
aws logs describe-log-streams --log-group-name /aws/application/api
aws logs tail /aws/application/api --follow
```

Or in the console: CloudWatch → Logs → Log groups → `/aws/application/api` →
select the log stream to see individual JSON log entries.

For querying, use CloudWatch Logs Insights against `/aws/application/api` — see
`examples/queries.txt` for the queries tested and their results.

## Screenshots

See `screenshots/`:
- `01-log-group.png` — log group details in the console
- `02-log-streams.png` — log stream with ingested events
- `03-log-insights.png` — Logs Insights query results
- `04-application-running.png` — application running and serving requests

## Challenges Faced and Solutions

- **Accidentally committed a 66MB `.rpm` file to git.** The CloudWatch agent
  installer was downloaded in the local repo folder instead of over SSH on the
  EC2 instance. Fixed by untracking the file, adding a `.gitignore`
  (`*.rpm`, `venv/`, `application.log`, etc.), and amending/force-pushing the
  single existing commit to remove it from history.

- **`requirements.txt` had unresolvable pinned versions** (e.g.
  `click==8.4.2`, which doesn't exist on PyPI) after a `pip freeze` was reused
  directly. Fixed by simplifying to unpinned top-level packages
  (`flask`, `structlog`, `boto3`) and letting pip resolve compatible
  sub-dependencies on the instance.

- **`scp` of the app folder was very slow.** Turned out the local `venv/`
  directory (thousands of small files) was nested inside `app/` and getting
  copied along with it. Fixed by copying only the source files
  (`server.py`, `requirements.txt`, `config.py`) and installing dependencies
  fresh on the instance instead.

- **Re-running the IAM setup script produced `EntityAlreadyExists` errors.**
  The role, policy, and instance profile had already been created in an
  earlier run. Verified the end state directly with
  `list-attached-role-policies`, `get-instance-profile`, and
  `describe-iam-instance-profile-associations` instead of re-creating
  anything, and confirmed everything was correctly wired.

- **SSH access** was restricted to a single security group rule scoped to my
  own IP (`/32`) rather than `0.0.0.0/0`, to avoid exposing port 22 to the
  internet.

## Reflection Questions

1. Why use structured logging instead of plain text?
Structured logging (JSON) makes each field (event, correlation_id, amount, etc.) individually searchable and filterable, so tools like CloudWatch Logs Insights can query and aggregate on them directly without regex. 
Plain text logs require string parsing to extract anything, making search and analysis far slower and error-prone.

2. What sensitive data should never be logged?
Sensitive data that should never appear in logs: passwords, API keys/tokens, credit card and other payment details, and personally identifying information like names, emails, phone numbers, or government IDs (e.g. SSN). 
Error messages and stack traces need care too, since they can accidentally leak this data if it was part of a failed request. 
Opaque identifiers like user_id, order_id, or correlation_id are fine to log. They're meant to be traceable for debugging and don't reveal a person's identity on their own.

3. How does correlation ID help debugging?
A correlation ID is attached to a request and passed along as it moves through each service, so every log line generated by that request.
 Across services it carries the same ID. This lets you filter on one ID and see the full path a request took, quickly narrowing down which service caused an issue rather than searching through unrelated logs from every service separately.

4. What retention period should you use?
Retention should match the environment and any compliance requirements, not be a single fixed number. Dev/lab environments (like this one) only need 15-30 days. 
It is enough to debug recent issues without paying to store data nobody will look at. 
Production typically needs longer (1-3 months or more), and regulated industries may require years of retention for audits (e.g. up to 7 years). 
For those longer requirements, logs are usually exported to cheaper storage (e.g. S3/Glacier) rather than kept in CloudWatch Logs at full price. 
This lab used 30 days, appropriate for a dev environment with no compliance obligation.

5. How can you reduce CloudWatch Logs costs?
CloudWatch Logs costs come down to three levers: what you log, how long you keep it, and how much you actually ship. 
Use appropriate log levels to avoid sending verbose DEBUG logs to CloudWatch in production, since that's often the biggest volume driver. 
Set retention based on actual need (see Q4) rather than defaulting to "never expire." And filter at the source to skip low-value noise (e.g. health-check spam), sample high-frequency events instead of logging every one, and use metric filters to pull out just a number (like a count or average) instead of storing and querying full raw log text when only a metric is needed.
