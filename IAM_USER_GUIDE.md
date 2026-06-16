# AWS IAM User — Explained Simply

You've been handed an **IAM user**. This guide explains *what that is*, *what to do first*,
and *how to actually use it* — assuming you know a little Linux and have seen an AWS instance,
but nothing about IAM. Take it step by step.

---

## Part 0 — Plain-English glossary (read first)

| Term | What it means (analogy) |
|---|---|
| **AWS Account** | The whole company workspace in Amazon's cloud — like the **office building**. |
| **Root user** | The **master key** to the whole building (the email that created the account). Almost nobody should use it day-to-day. **Not you.** |
| **IAM** (Identity and Access Management) | The **security desk** that hands out ID badges and decides which rooms each badge opens. |
| **IAM user** (you) | **Your personal ID badge** for the building. It identifies you and controls what you're allowed to touch. |
| **Credentials** | The ways you prove the badge is yours. There are **two kinds** (see below). |
| **Console password** | Lets you log into the AWS **website** (the "AWS Console") in a browser. |
| **Access key** (Access Key ID + Secret Access Key) | A username/password pair for the **command line / code**, not the website. Like a key for tools, not for the front door. |
| **MFA** (Multi-Factor Auth) | A second proof of identity — a 6-digit code from your phone. Strongly recommended. |
| **Policy** | The **rules on your badge**: "can open room A and B, not room C." Decides your permissions. |
| **Permissions** | What your badge is actually allowed to do (e.g. "view EC2 instances," "read this S3 bucket"). |
| **EC2 instance** | A **server (a computer)** running in AWS. The thing you SSH into. |
| **Region** | **Which city's data center** your stuff lives in (e.g. `us-east-1`, `ap-southeast-1`). Things in one region don't show up if you're looking at another. |
| **AWS CLI** | A **command-line tool** on your Linux machine that talks to AWS, so you can do things by typing instead of clicking. |
| **ARN** | A long unique **address/ID** for an AWS thing (you'll see these; just know it's an identifier). |

The one-sentence summary: **An IAM user is your personal badge into someone's AWS account, and your permissions (policies) decide which doors it opens.**

---

## Part 1 — What you were probably given

When someone creates an IAM user for you, they hand over **one or both** of these:

1. **Console sign-in info** (to use the website):
   - An **Account ID** or a special **sign-in URL** (looks like `https://123456789012.signin.aws.amazon.com/console`)
   - Your **username**
   - A **temporary password** (you'll likely be told to change it on first login)

2. **Access keys** (to use the command line / AWS CLI):
   - **Access Key ID** (looks like `AKIA...`)
   - **Secret Access Key** (a long random string — shown **only once**, treat it like a password)

> ⚠️ **Never share or commit your Secret Access Key.** Don't paste it into chats, code, or GitHub.
> If it ever leaks, tell your admin to deactivate it immediately.

If you only got *some* of these, that's normal — ask your admin which kind of access you have.

---

## Part 2 — First things to do (in order)

### Step 1 — Log into the AWS Console (the website)
1. Open the **sign-in URL** your admin gave you (or go to `https://console.aws.amazon.com` and
   choose **"IAM user,"** then enter the **Account ID**).
2. Enter your **username** and **temporary password**.
3. If asked, **set a new password**. Use a strong, unique one (a password manager helps).

### Step 2 — Turn on MFA (do this early — it's your seatbelt)
1. Install an authenticator app on your phone (Google Authenticator, Authy, Microsoft Authenticator).
2. In the Console, top-right click your name → **Security credentials**.
3. Find **Multi-factor authentication (MFA)** → **Assign MFA device**.
4. Choose **Authenticator app**, scan the QR code with your phone app, type in **two** consecutive
   6-digit codes when asked, and confirm.
5. From now on, logging in asks for your password **and** the phone code. Good.

### Step 3 — Find out what you're allowed to do
- In the Console, go to the **IAM** service → **Users** → click **your username** → **Permissions** tab.
- You'll see the **policies** attached to you. This is the list of what your badge can open.
- Don't worry about reading the raw rules — just know: **if something says "access denied" later,
  it usually means your policy doesn't include that permission.** Ask your admin to add it.

### Step 4 — Note your Region
- Top-right of the Console shows a region (e.g. **N. Virginia / us-east-1**).
- If you don't see "your" servers, you're probably looking at the **wrong region** — switch it
  to the one your team uses. Ask your admin which region your stuff is in.

---

## Part 3 — Set up the AWS CLI on your Linux machine

This lets you control AWS by typing commands. You'll need your **Access Key ID** and
**Secret Access Key** from Part 1.

### Step 1 — Install the CLI
```bash
# Download and install the official AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version          # should print a version number
```

### Step 2 — Connect it to your account
```bash
aws configure
```
It asks four things — paste/enter each:
```
AWS Access Key ID     : AKIA...           (your access key id)
AWS Secret Access Key : ****************   (your secret key)
Default region name   : us-east-1         (the region your team uses)
Default output format : json              (just type: json)
```
This saves your settings in `~/.aws/credentials` and `~/.aws/config` on your machine.

> 🔒 That file holds your secret key. Keep your laptop secure and **never** copy `~/.aws/credentials`
> into a repo.

### Step 3 — Prove it works
```bash
aws sts get-caller-identity
```
If it prints your user's account number and ARN (your badge's ID), you're connected. 🎉

A few harmless commands to explore (you can only see what your policy allows):
```bash
aws ec2 describe-instances --output table      # list servers in your region
aws s3 ls                                       # list storage buckets you can see
```
If any say **"AccessDenied,"** that's just your permissions — not a mistake on your part.

---

## Part 4 — Connecting to an EC2 instance (a server)

You said you've seen AWS instances — here's the clean way in.

### Option A — SSH with a key file (classic)
1. Get the **private key file** (`something.pem`) from your admin, and the server's **public
   address** (Public IP or DNS name).
2. Lock down the key file's permissions (SSH refuses loose ones):
   ```bash
   chmod 400 mykey.pem
   ```
3. Connect (the username depends on the server's OS):
   ```bash
   ssh -i mykey.pem ec2-user@<public-ip>     # Amazon Linux
   ssh -i mykey.pem ubuntu@<public-ip>       # Ubuntu
   ```
   If it hangs or refuses, the server's **Security Group** (its firewall) may not allow your
   IP on port 22 — ask your admin to open it for you.

### Option B — Session Manager (no SSH key, no open ports — often preferred)
If your account uses **AWS Systems Manager**, you can get a shell **without** a key file or any
open inbound port:
```bash
aws ssm start-session --target i-0123456789abcdef0
```
(`i-0123...` is the **instance ID**, which you can see in the EC2 console or from
`aws ec2 describe-instances`.) Ask your admin if this is set up — it's the safer modern way in.

---

## Part 5 — Safety rules (short and important)

- **Protect your Secret Access Key like a password.** Never share, never commit to git.
- **Turn on MFA** (Part 2, Step 2).
- **Don't use the root user** — that's not you, and nobody should for daily work.
- **Least privilege is normal:** you'll only have the permissions you need. "Access denied" usually
  just means "ask the admin to grant that."
- **Rotate keys** if one ever leaks: deactivate the old one in IAM → Security credentials, create a new one.
- **Log out / lock your laptop** — your badge is only as safe as your machine.

---

## Quick cheat sheet

```bash
aws configure                 # set up credentials (one time)
aws sts get-caller-identity   # "who am I?" — confirm you're connected
aws ec2 describe-instances    # list servers you can see (current region)
aws s3 ls                     # list storage you can see
aws ssm start-session --target i-xxxx   # shell into a server, no SSH key
ssh -i key.pem ec2-user@IP    # classic SSH into a server
```

## When you're stuck, ask your admin these exact questions
1. "Which **region** is our stuff in?"
2. "Does my IAM user have permission to **do X**?" (when you hit Access Denied)
3. "Can I get in via **Session Manager**, or do I need an **SSH key + open port 22**?"
4. "Can you confirm my **MFA** is required and my **access key** is active?"

---

### TL;DR
An **IAM user** is your personal badge into an AWS account. First: **log into the website, set MFA,
check your permissions, note your region.** Then: **install the AWS CLI and run `aws configure`** to
control things from Linux. To reach a server, use **`ssm start-session`** (no key) or **`ssh -i key.pem`**.
Guard your secret key, and "Access Denied" just means "ask the admin for that permission."
