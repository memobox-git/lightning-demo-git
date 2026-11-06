# Setup: everything to complete before building the pipeline

This is the groundwork for the Olist Lightning Pipeline. Finish every section
here first; the pipeline steps (S3 → bronze → silver → gold → dashboard) assume
all of it exists. Budget about 60 minutes the first time.

What you end up with:

```
Claude Code cloud session ("demo" environment)
   │  HTTPS only
   ├──> GitHub repo            code, one commit per step
   ├──> Kaggle API             Olist dataset (token saved on the environment)
   ├──> S3 bucket              raw zone + gold snapshots
   └──> AWS Lambda  ───────────┐  runs the DB steps inside the VPC
                               ▼
                       RDS PostgreSQL  (bronze, silver, gold, meta)
                               ▲
   DBeaver on your laptop ─────┘  port 5432, your IP only
```

Why the Lambda: a Claude cloud session can only make HTTPS calls through a
proxy. A PostgreSQL connection (port 5432) never leaves the sandbox. So the
database steps run on a small machine inside AWS that Claude starts and reads
over HTTPS. If you run Claude Code on your laptop instead, skip section 7 and
run the scripts directly.

---

## 1. Accounts

| Account | Needed for | Notes |
|---|---|---|
| AWS | RDS, S3, Lambda, Secrets Manager | Any account; costs below |
| GitHub | the repo | Connected to Claude (claude.ai → Settings → Connectors → GitHub) |
| Kaggle | the Olist dataset | Free. Settings → API → Generate New Token |
| Claude | Claude Code cloud sessions | Pro or Max plan |

## 2. AWS IAM user and keys

Create an IAM user for the demo and one access key pair.

```bash
aws iam create-user --user-name lightning-demo --tags Key=project,Value=olist-lightning
aws iam attach-user-policy --user-name lightning-demo --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam create-access-key --user-name lightning-demo
```

Administrator access is the fast path for a demo. The least the pipeline needs
is: RDS, S3, EC2 (security groups, VPC endpoints), Lambda, IAM (create one
role), Secrets Manager read.

Keep the key pair for step 3. Never paste it into a chat, a file in the repo,
or a commit message.

## 3. Claude cloud environment "demo"

In claude.ai → Code → environment menu → New environment:

| Setting | Value |
|---|---|
| Name | `demo` |
| Repository | your fork of this repo |
| Network | Limited (package managers allowed) |
| Environment variables | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` from step 2 |

Do **not** set `AWS_PROFILE` here; no profile file exists in the sandbox, and
the keys are picked up from the variables. On a laptop, use a profile named
`lightning-demo` instead (`aws configure --profile lightning-demo`).

Region is always `us-east-1`. Every resource carries the tag
`project=olist-lightning`.

## 4. Kaggle token on the environment

The Kaggle CLI (v2) talks to **api.kaggle.com**; older docs say www.kaggle.com.
Save the same token as a network secret for both hosts. From a session, ask
Claude: "Save my Kaggle API token for www.kaggle.com and api.kaggle.com". A
secure form opens; the value never passes through the chat.

In commands, use the placeholder `KAGGLE_API_TOKEN=proxy-injected`; the proxy
swaps in the real token.

Test:

```bash
KAGGLE_API_TOKEN=proxy-injected kaggle datasets files olistbr/brazilian-ecommerce
```

## 5. RDS PostgreSQL

Smallest standard instance, password managed by Secrets Manager (nobody ever
sees it), not public at first.

```bash
aws rds create-db-instance \
  --db-instance-identifier lightning-demo-rds \
  --engine postgres --engine-version 18.3 \
  --db-instance-class db.t4g.micro \
  --allocated-storage 20 --storage-type gp3 --storage-encrypted \
  --master-username postgres --manage-master-user-password \
  --db-name olist \
  --no-multi-az --no-publicly-accessible \
  --backup-retention-period 1 --no-deletion-protection \
  --tags Key=project,Value=olist-lightning
aws rds wait db-instance-available --db-instance-identifier lightning-demo-rds
```

Record two values for `config.env`:

```bash
aws rds describe-db-instances --db-instance-identifier lightning-demo-rds \
  --query 'DBInstances[0].[Endpoint.Address,MasterUserSecret.SecretArn]' --output text
```

`DB_HOST` is the endpoint. `DB_SECRET_NAME` is the part of the secret ARN
after `secret:` without the random 6-character suffix (looks like
`rds!db-<uuid>`).

### Open it to your laptop (DBeaver)

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
MYIP=$(curl -s https://checkip.amazonaws.com)
SG=$(aws ec2 create-security-group --group-name lightning-demo-rds-access \
  --description "Postgres access to lightning-demo-rds" --vpc-id $VPC --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG \
  --ip-permissions "IpProtocol=tcp,FromPort=5432,ToPort=5432,IpRanges=[{CidrIp=$MYIP/32}]"
aws rds modify-db-instance --db-instance-identifier lightning-demo-rds \
  --publicly-accessible --vpc-security-group-ids $SG --apply-immediately
```

DBeaver: host = endpoint, port 5432, database `olist`, user `postgres`,
SSL `require`. Password: AWS console → Secrets Manager → the `rds!db-…` secret
→ Retrieve secret value.

## 6. S3 bucket

Globally unique name starting with `olist-lightning-`.

```bash
B=olist-lightning-<yourname>-<random>
aws s3api create-bucket --bucket $B
aws s3api put-public-access-block --bucket $B \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-versioning --bucket $B --versioning-configuration Status=Enabled
aws s3api put-bucket-tagging --bucket $B --tagging 'TagSet=[{Key=project,Value=olist-lightning}]'
```

The bucket name goes in `config.env` only. Never commit it.

## 7. Lambda runner (cloud sessions only)

Needed because the cloud sandbox cannot open port 5432. Four pieces.

**a. Security groups.** One for the Lambda, one for the VPC endpoints; let the
Lambda reach RDS and the endpoints.

```bash
LSG=$(aws ec2 create-security-group --group-name olist-lightning-lambda --description "pipeline lambda" --vpc-id $VPC --query GroupId --output text)
ESG=$(aws ec2 create-security-group --group-name olist-lightning-endpoints --description "vpc endpoints" --vpc-id $VPC --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $ESG --ip-permissions "IpProtocol=tcp,FromPort=443,ToPort=443,UserIdGroupPairs=[{GroupId=$LSG}]"
aws ec2 authorize-security-group-ingress --group-id $SG  --ip-permissions "IpProtocol=tcp,FromPort=5432,ToPort=5432,UserIdGroupPairs=[{GroupId=$LSG}]"
```

**b. VPC endpoints.** The Lambda has no internet; it reaches S3 through a free
gateway endpoint and Secrets Manager through an interface endpoint (~$7/month).

```bash
RTB=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPC --query 'RouteTables[0].RouteTableId' --output text)
SUBNET=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC Name=availability-zone,Values=us-east-1a --query 'Subnets[0].SubnetId' --output text)
aws ec2 create-vpc-endpoint --vpc-id $VPC --vpc-endpoint-type Gateway --service-name com.amazonaws.us-east-1.s3 --route-table-ids $RTB
aws ec2 create-vpc-endpoint --vpc-id $VPC --vpc-endpoint-type Interface --service-name com.amazonaws.us-east-1.secretsmanager \
  --subnet-ids $SUBNET --security-group-ids $ESG --private-dns-enabled
```

**c. IAM role.** Read the bucket, read the one secret, write logs, attach to
the VPC.

```bash
SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier lightning-demo-rds --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text)
aws iam create-role --role-name olist-lightning-lambda --assume-role-policy-document \
  '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name olist-lightning-lambda --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
aws iam put-role-policy --role-name olist-lightning-lambda --policy-name olist-lightning-data --policy-document "{
 \"Version\":\"2012-10-17\",\"Statement\":[
  {\"Effect\":\"Allow\",\"Action\":[\"s3:GetObject\",\"s3:PutObject\"],\"Resource\":\"arn:aws:s3:::$B/*\"},
  {\"Effect\":\"Allow\",\"Action\":\"s3:ListBucket\",\"Resource\":\"arn:aws:s3:::$B\"},
  {\"Effect\":\"Allow\",\"Action\":\"secretsmanager:GetSecretValue\",\"Resource\":\"$SECRET_ARN\"}]}"
```

**d. The function.** Settings are passed as environment variables (same names
as `config.env`; no secrets). Start it with a stub; the pipeline's `deploy.sh`
replaces the code.

```bash
echo 'def lambda_handler(e, c): return {"ok": False, "error": "no code deployed yet"}' > handler.py && zip stub.zip handler.py
aws lambda create-function --function-name olist-lightning-pipeline --runtime python3.13 \
  --handler handler.lambda_handler --zip-file fileb://stub.zip \
  --role arn:aws:iam::<ACCOUNT_ID>:role/olist-lightning-lambda \
  --timeout 900 --memory-size 2048 --ephemeral-storage Size=2048 \
  --vpc-config SubnetIds=$SUBNET,SecurityGroupIds=$LSG \
  --environment "Variables={S3_BUCKET=$B,DB_HOST=<endpoint>,DB_PORT=5432,DB_NAME=olist,DB_USER=postgres,DB_SECRET_NAME=<rds!db-...>,BI_USER=bi_reader,OWNER=<you>}" \
  --tags project=olist-lightning
```

First creation takes a few minutes while the VPC network interface attaches.

## 8. The repo and `config.env`

```bash
git clone <your fork> && cd claude-code-data-pipeline
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
cp config.env.example config.env
```

Fill `config.env`:

| Key | Value |
|---|---|
| `AWS_PROFILE` | `lightning-demo` (laptop only; unset in a cloud session) |
| `AWS_REGION` | `us-east-1` |
| `OWNER` | your name |
| `S3_BUCKET` | from step 6 |
| `DB_HOST`, `DB_SECRET_NAME` | from step 5 |

`config.env` is git-ignored. It is the only place the bucket name lives.

## 9. Checklist before the first pipeline step

- [ ] `aws sts get-caller-identity` returns `user/lightning-demo`
- [ ] `kaggle datasets files olistbr/brazilian-ecommerce` lists 9 files
- [ ] RDS status `available`; DBeaver connects and `olist` is empty
- [ ] `aws s3 ls s3://$B/` returns nothing (empty bucket)
- [ ] Lambda invoke returns the stub message
- [ ] `config.env` filled; `git status` does not list it

## 10. Cost and teardown

| Item | Approx. monthly |
|---|---|
| RDS db.t4g.micro + 20 GB | $12–15 (free tier may cover it) |
| Secrets Manager endpoint | $7 |
| Lambda, S3, secret | under $1 |

Teardown, in order: delete the Lambda, the two VPC endpoints, the RDS instance
(`--skip-final-snapshot`), empty and delete the bucket, delete the three
security groups, the IAM role and the IAM user.
