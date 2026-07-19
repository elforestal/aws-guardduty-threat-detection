# Threat Detection with GuardDuty

**Author:** Edith Forestal
**GitHub:** [github.com/elforestal](https://github.com/elforestal) ·
**LinkedIn:** [linkedin.com/in/forestal](https://linkedin.com/in/forestal)

---

<img width="1472" height="1400" alt="image" src="https://github.com/user-attachments/assets/0aa03013-6383-4582-af71-747b5b204b37" />


## Introducing Today's Project!

### Tools and concepts

This project spanned the full security lifecycle — deploying a vulnerable environment, exploiting it as an attacker, then detecting and analyzing the attack as a defender. It combines infrastructure as code, hands-on offensive techniques, and cloud-native threat detection to show how a web-layer flaw can escalate into a full cloud compromise, and how that compromise gets caught.

**Services used:**

- ☁️ **CloudFormation** — deploy the whole environment as code
- 🖥️ **EC2, VPC, S3, CloudFront** — the vulnerable web app infrastructure
- 💻 **CloudShell + AWS CLI** — running the attack
- 🛡️ **GuardDuty** (incl. Malware Protection for S3) — threat detection

**Attack chain (web layer → cloud):**

- 💉 **SQL injection** — bypass admin authentication
- ⚙️ **Command injection** — abuse the EC2 instance metadata service to leak IAM role credentials
- 🔑 **Credential exfiltration** — use stolen credentials from a separate account to steal data from S3

**Defensive concepts:**

- 📊 **Anomaly detection** — GuardDuty flags instance credentials used from an unexpected account
- 🔍 **Finding analysis** — reading a finding to scope an incident
- 🦠 **Malware Protection** — content-based detection at the storage layer

**Security principles reinforced:** defense in depth · least privilege · input sanitization · credential hygiene

### Project reflection

This project took me approximately 2 hours, though a good chunk went into troubleshooting rather than the lab steps.

The most challenging part was a GuardDuty deployment failure unrelated to the walkthrough — my CloudFormation stack kept rolling back on the detector with a 403 "needs a subscription" error, and I had to rule out region, permissions, and the template before tracing it to an account-level free-plan restriction and resolving it as account owner.

It was most rewarding to see the chain pay off end to end — running a real attack from SQL injection through credential exfiltration to S3 data theft, then switching to the defender's seat and watching GuardDuty surface it as a High-severity instance credential exfiltration finding. Seeing detection engineering work from attack to alert was the part that made it click.

I did this project to extend my portfolio from secure-by-default provisioning into detection engineering — I've been building out cloud security projects (KMS, IAM, Terraform) with a security-engineer framing, and GuardDuty was the natural next step to show I can reason about threats from both the attacker's and defender's side, not just build infrastructure.

It met that goal well. Running the full attack chain — SQL injection, command injection, IAM credential exfiltration, S3 data theft — and then analyzing exactly how GuardDuty's anomaly detection surfaced it gave me hands-on grounding in how detection actually works, plus a real troubleshooting story from resolving the account-level GuardDuty block. That combination of offensive understanding, defensive analysis, and practical problem-solving is exactly what I want to demonstrate for Senior Cloud Security Engineer and DevSecOps roles.

---

## Project Setup

To set up for this project, I deployed a CloudFormation template that launches a deliberately vulnerable web app — a copy of the OWASP Juice Shop — along with the full environment needed to attack and monitor it.

The three main components are the web app infrastructure (an EC2 web server hosting the app inside a purpose-built VPC with its own subnets, security group, load balancer, and CloudFront distribution, isolated from the default VPC to contain blast radius),

an S3 bucket holding a simulated sensitive file that stands in as the data breach target,

and a GuardDuty detector enabled for continuous threat detection so I can later verify whether the attack chain is caught.

The web app deployed is called the OWASP Juice Shop — an intentionally vulnerable application built for security training, packed with realistic flaws on purpose.

To practice my GuardDuty skills, I will step into the attacker's role and run a full exploitation chain against it: using SQL injection to bypass the admin login, command injection to make the EC2 web server leak its own IAM credentials, and those stolen credentials to exfiltrate sensitive data from the S3 bucket.

Then I'll switch back to the defender's seat and check whether GuardDuty detected the credential misuse, so I can analyze the finding it raises and understand how the attack surfaced from a detection standpoint.

GuardDuty is AWS's managed threat detection service that continuously monitors account activity, network traffic, and API logs (like CloudTrail) and uses anomaly detection and machine learning to flag suspicious behavior — surfacing it as prioritized findings without any agents to deploy or rules to hand-write.

In this project, it will act as the defender's line of sight: after I attack the web app and exfiltrate the EC2 instance's IAM credentials, GuardDuty should detect the credentials being used from an unexpected location and raise a finding, letting me confirm the attack chain was caught and analyze exactly what it saw.

![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-guardduty_n1o2p3q4)

---
---

## 🔴 Challenge: GuardDuty Detector Failed to Deploy

Before I could run the project, my CloudFormation deploys rolled back repeatedly, failing on the `AWS::GuardDuty::Detector` resource with:

`403 — "The AWS Access Key Id needs a subscription for the service."`

I ruled out the usual suspects methodically:

- 🌍 **Region** — failed identically in us-east-2 and us-east-1, so not regional
- 🔑 **Permissions** — the error was a service-subscription 403, not an IAM `AccessDenied`, and I was operating as an admin identity
- 📄 **Template** — every other resource validated cleanly; only the detector failed

The root cause was account-level: the **AWS Free account plan restricts GuardDuty subscription**. I resolved it as account owner by upgrading to pay-as-you-go, confirmed eligibility in the GuardDuty console, then returned to my IAM admin identity to let the template manage the detector lifecycle — never running the lab as root.

**Takeaway:** some AWS services require account-level enablement before IaC can provision them. Diagnosing this meant reading the actual API error rather than assuming a template bug — and keeping identity discipline throughout: root only for the account-owner action, IAM admin for everything else.
<img width="1874" height="752" alt="Screenshot 2026-07-19 142336" src="https://github.com/user-attachments/assets/65f10894-bac1-46bf-ade2-bdf9665eb595" />

---
---

## SQL Injection

The first attack I performed on the web app is SQL injection, which means inserting malicious SQL code into an input field so it gets executed as part of the application's database query rather than treated as plain data — here, entering `' or 1=1;--` in the login field rewrites the authentication query so it always evaluates to true, bypassing the password check entirely.

SQL injection is a security risk because it stems from an app trusting unsanitized user input, and it can let an attacker bypass authentication, read or exfiltrate sensitive data, or even alter the database structure — which is why input validation, parameterized queries, and least-privilege database accounts are core defenses against it.

My SQL injection attack involved entering the string `' or 1=1;--` into the email field of the login page, then any value in the password field, which bypassed authentication and logged me straight into the admin account.

This means the app inserted my input directly into its login SQL query without sanitizing it: the `'` closes the expected string, `or 1=1` adds a condition that's always true so the query returns a valid user regardless of credentials, and `--` comments out the rest of the query (including the password check) so it never runs — a textbook sign the app is building queries by concatenating raw input instead of using parameterized statements.



![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-guardduty_h1i2j3k4)

---

## Command Injection

Next, I used command injection, which is a vulnerability where an attacker inserts operating-system or runtime commands into an input field and the server executes them, instead of just storing the input as data — here, I pasted a JavaScript payload into the profile's username field and the EC2 web server ran it, forcing it to query its own instance metadata service and write its IAM credentials to a public file.

The Juice Shop web app is vulnerable to this because it doesn't sanitize input before processing it — it passes what should be a harmless username straight into a function that executes code, so the server can't tell the difference between legitimate data and an attacker's commands, which is why input sanitization and least-privilege instance roles are the key defenses.

To run command injection, I pasted a JavaScript payload into the username field on the admin profile page and selected Set Username, which the unsanitized web server executed as code instead of storing it as text — confirmed when the username defaulted to [object Object], signaling a new JavaScript object had been created.
The script will force the EC2 web server to request a session token from its instance metadata service, use that token to retrieve the instance's IAM role credentials, and write them to a publicly accessible credentials.json file in the app's assets folder — turning a simple profile field into a credential exfiltration point that anyone on the internet can reach.

![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-guardduty_t3u4v5w6)

---

## Attack Verification

To verify the attack's success, I navigated to the public `credentials.json` file my injected script created — at `d2nan5lifr79h.cloudfront.net/assets/public/credentials.json` — and confirmed the EC2 instance's IAM credentials were sitting there, exposed to anyone who visits that URL.

The credentials page showed me a full set of temporary AWS credentials — `AccessKeyId`, `SecretAccessKey`, `Token`, and an `Expiration` timestamp — which proved the command injection worked end to end: the server queried its own instance metadata and leaked its IAM role credentials to a publicly reachable file, giving an attacker everything needed to authenticate to the AWS environment as the web app.

![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-guardduty_x7y8z9a0)

---

## Using CloudShell for Advanced Attacks

The attack continues in CloudShell, because it gives me a realistic way to simulate an outside attacker: CloudShell sessions run under a temporary AWS account ID that's separate from the EC2 instance the stolen credentials belong to, so using those credentials here looks like they're being wielded from a different account entirely.

That separation is exactly what makes the activity anomalous — credentials issued to my EC2 instance role are suddenly being used by a different account ID to access resources, which strays from the instance's normal behavior and is precisely the credential-exfiltration pattern GuardDuty's anomaly detection is built to flag.

In CloudShell, I used `wget` to download the exposed `credentials.json` file from the web app's public URL into my CloudShell environment, pulling the stolen IAM credentials out of the browser and onto the command line where I could actually use them.

Next, I ran a command using `cat` and `jq` to display and parse that file — `cat` prints the file's raw contents, and piping it through `jq` formats the JSON so the `AccessKeyId`, `SecretAccessKey`, and `Token` are cleanly readable — confirming I had a valid, usable set of credentials before configuring them into an AWS CLI profile for the next stage of the attack.

I then set up a profile, called `stolen`, to load the exfiltrated IAM credentials — the `AccessKeyId`, `SecretAccessKey`, and session `Token` — into the AWS CLI so I could authenticate to the AWS environment as the compromised EC2 instance role.

I had to create a new profile because CloudShell's default profile already inherits my own IAM user's permissions, and running the attack under those would just look like normal, authorized activity. Isolating the stolen credentials in a separate `stolen` profile means my commands are explicitly authenticated as the EC2 instance's role from a different account context — the anomalous credential use that makes the exfiltration both realistic and detectable by GuardDuty.

![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-guardduty_j9k0l1m2)

---

## GuardDuty's Findings

After performing the attack, GuardDuty reported a finding within 5 minutes — anomaly-based detection isn't instant, since it needs time to observe the activity, compare it against the instance's baseline behavior, and confirm the deviation before raising an alert.

UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.InsideAWS

GuardDuty's detailed finding reported that IAM credentials belonging exclusively to my EC2 instance role were used from a remote AWS account — a High-severity InstanceCredentialExfiltration.InsideAWS event, meaning the instance's credentials had almost certainly been stolen and were being wielded from a different account than the one they were issued to.

The finding broke the attack down into actionable detail: the affected Resource was the EC2 instance role used to reach the environment, the Action showed an S3 object being retrieved (my exfiltration of the secret file), and the Location and IP tied the activity to my AWS Region at the time I ran the stolen-credential commands in CloudShell.

![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-guardduty_v1w2x3y4)

---

## Extra: Malware Protection

For my project extension, I enabled GuardDuty Malware Protection for S3 on the bucket I'd just exfiltrated data from — configuring it to scan newly uploaded objects and letting AWS create a scoped service role that grants GuardDuty only the permissions needed to read and scan those objects, keeping the integration least-privilege.

To test Malware Protection, I uploaded an EICAR test file into the protected S3 bucket — a standardized dummy file that antivirus and malware-detection tools are built to recognize as a threat, letting me trigger GuardDuty's scanning without touching anything actually malicious.

The uploaded file won't actually cause damage because it isn't real malware — it's a harmless text string defined by the European Institute for Computer Antivirus Research specifically for safely testing detection systems. It carries no payload and can't execute or spread, so it verifies that GuardDuty's Malware Protection scans new uploads and flags threats exactly as it should, without ever putting the environment at real risk.

Once I uploaded the file, GuardDuty instantly triggered a malware finding — scanning the new S3 object, matching it against known malware signatures, recognizing the EICAR test file as a threat, and raising a finding that flagged the bucket and the malicious object.

This verified that Malware Protection was working end to end: GuardDuty automatically scans objects as they land in the bucket and alerts on malicious content without any manual intervention, confirming I now have content-based threat detection at the storage layer layered on top of the credential-anomaly detection I tested earlier — defense in depth across two distinct threat types in the same environment.

![Image](http://nextwork.ai/passionate_rose_serene_emblica/uploads/aws-security-guardduty_sm42x3y4)

---
