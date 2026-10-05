---
title: "GraphRAG in Neo5j on AWS"
subtitle: "Conversational AI for Rural Economic Development Data"
author: "Mapping & Data Analytics at the Center on Rural Innovation"
execute:
  echo: true
  output: true
  message: false
  warning: false
format:
  gfm:
    output-file: README.md
    standalone: false
---

So first we need to setup a virtual machine that can host the Neo4j graph database. This will happen in two parts. First, we'll provision the basic EC2 instance in a state that Neo4j can be installed, but before the actual installation, we will create an image (AMI) of this base state that we can use for later deployments. Second, on this base server we will proceed to actually install, configure and use Neo4j in order to validate the viability of this system.

### I. Baking the Ubuntu 26.04 Neo4j CE AMI

Run steps 1–6 and 8–10 in one terminal on your Mac, because later steps reuse the shell variables. Step 7 runs on the builder instance. You'll need the AWS CLI and the Session Manager plugin (`brew install --cask session-manager-plugin`).

#### Step 1: Region and profile

```bash
export AWS_REGION=us-east-1
# export AWS_PROFILE=<the profile you use for `npx ampx sandbox`>
```

#### Step 2: Find Canonical's current Ubuntu 26.04 AMI

```bash
export UBUNTU_AMI=$(aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/26.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --query Parameter.Value --output text)
aws ec2 describe-images --image-ids "$UBUNTU_AMI" \
  --query 'Images[0].[ImageId,Name,RootDeviceName]' --output text
```

The root device should read `/dev/sda1`, which is the name the CDK `blockDevices` entry uses.

#### Step 3: Create an IAM role for the builder (one time; keep it for future rebakes)

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export KMS_ALIAS=alias/encryption-in-transit

export BUILDER_ROLE_ARN=$(aws iam create-role --role-name neo4j-ami-builder \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  --query 'Role.Arn' --output text)

aws iam attach-role-policy --role-name neo4j-ami-builder \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

export SESSION_KMS_POLICY=$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "kms:Decrypt",
    "Resource": "arn:aws:kms:${AWS_REGION}:${ACCOUNT_ID}:key/*",
    "Condition": { "ForAnyValue:StringEquals": { "kms:ResourceAliases": "${KMS_ALIAS}" } }
  }]
}
EOF
)
aws iam put-role-policy --role-name neo4j-ami-builder --policy-name session-manager-kms \
  --policy-document "$SESSION_KMS_POLICY"

export BUILDER_PROFILE_ARN=$(aws iam create-instance-profile --instance-profile-name neo4j-ami-builder \
  --query 'InstanceProfile.Arn' --output text)
aws iam add-role-to-instance-profile --instance-profile-name neo4j-ami-builder --role-name neo4j-ami-builder

echo "$BUILDER_ROLE_ARN"
echo "$BUILDER_PROFILE_ARN"
```

Wait about 15 seconds before step 5 so the new instance profile can propagate.

#### Step 4: Pick a subnet with internet access for the builder

The builder downloads packages, so it can't go in the isolated CORI subnets.

```bash

export CORI_VPC=vpc-08f5e17f5b75ccee9
export BUILD_SUBNET=subnet-0753143006c468f97   # public subnet (internet access), us-east-1a

# Route table explicitly associated with the subnet. Empty output means it uses the VPC's main route table.
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values="$BUILD_SUBNET" \
  --query 'RouteTables[].[RouteTableId,Associations[].[RouteTableAssociationId,SubnetId,Main],Routes[].[DestinationCidrBlock,GatewayId,NatGatewayId]]' \
  --output json

# If that was empty, the main route table:
aws ec2 describe-route-tables --filters Name=vpc-id,Values="$CORI_VPC" Name=association.main,Values=true \
  --query 'RouteTables[].[RouteTableId,Routes[].[DestinationCidrBlock,GatewayId,NatGatewayId]]' --output json
```

#### Step 5: Launch the builder

It has no key pair and no inbound rules; you reach it only through Session Manager.

```bash
BUILDER_ID=$(aws ec2 run-instances \
  --image-id "$UBUNTU_AMI" \
  --instance-type t3a.large \
  --subnet-id "$BUILD_SUBNET" \
  --associate-public-ip-address \
  --iam-instance-profile Name=neo4j-ami-builder \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":24,"VolumeType":"gp3","Encrypted":true,"DeleteOnTermination":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=neo4j-ami-builder}]' \
  --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-running --instance-ids "$BUILDER_ID"
aws ssm describe-instance-information --filters "Key=InstanceIds,Values=$BUILDER_ID" \
  --query 'InstanceInformationList[0].PingStatus' --output text
```

Re-run the last command until it prints `Online`.

#### Step 6: Connect to the builder

```bash
aws ssm start-session --target "$BUILDER_ID"
```

#### Step 7: On the builder, write and run the bake script

The script runs under `nohup` with a log file, so it keeps going if the session drops. The version is pinned in the script. The script prints the available versions (from `apt-cache madison`) before installing, so if the pinned one doesn't exist, the log shows what to change it to.

```bash
sudo -i
cat > /root/bake-neo4j-ce.sh <<'EOF'
#!/bin/bash
# Bakes Neo4j Community Edition onto Ubuntu 26.04 for the cori.agent Neo4j AMI.
# Run as root on the builder instance.
set -euxo pipefail
export DEBIAN_FRONTEND=noninteractive
export NEO4J_VERSION='1:2026.09.0'   # epoch:version, as listed by apt-cache madison neo4j
export NEO4J_UID=500                 # fixed uid/gid so retained data volumes stay readable across rebakes

# Sets a single-valued setting in neo4j.conf: rewrites the commented or active line, or appends it.
# Do NOT use it for server.jvm.additional. That key is on many lines of the shipped file, and this
# helper would rewrite every one of them (the package ships active JVM options we must keep).
set_neo4j_conf() {
  sed -i "s|^#\?$1=.*|$1=$2|" /etc/neo4j/neo4j.conf
  grep -q "^$1=" /etc/neo4j/neo4j.conf || echo "$1=$2" >> /etc/neo4j/neo4j.conf
}

# 1. Patch the OS; Java 21 and tools
apt-get update
apt-get -y full-upgrade
apt-get install -y ca-certificates curl gnupg unzip openjdk-21-jre postgresql-client

# 2. neo4j user/group with a fixed id, created before the package would create them
getent group neo4j >/dev/null || groupadd --system --gid "$NEO4J_UID" neo4j
getent passwd neo4j >/dev/null || useradd --system --uid "$NEO4J_UID" --gid neo4j \
  --home-dir /var/lib/neo4j --no-create-home --shell /bin/bash neo4j

# 3. Neo4j apt repository and the pinned Community Edition package, held
install -d -m 0755 /etc/apt/keyrings
curl -fsSL https://debian.neo4j.com/neotechnology.gpg.key | gpg --dearmor --yes -o /etc/apt/keyrings/neotechnology.gpg
echo 'deb [signed-by=/etc/apt/keyrings/neotechnology.gpg] https://debian.neo4j.com stable latest' > /etc/apt/sources.list.d/neo4j.list
apt-get update
apt-cache madison neo4j | head -10 || true
apt-get install -y "neo4j=$NEO4J_VERSION"
apt-mark hold neo4j cypher-shell
java -version

# 4. AWS CLI v2 (UserData reads the admin password from Secrets Manager)
curl -fsSL https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o /tmp/awscliv2.zip
unzip -q -o /tmp/awscliv2.zip -d /tmp
/tmp/aws/install --update
rm -rf /tmp/aws /tmp/awscliv2.zip
/usr/local/bin/aws --version

# 5. SSM agent: snap or deb package, depending on the image
if snap list amazon-ssm-agent >/dev/null 2>&1; then
  systemctl is-enabled snap.amazon-ssm-agent.amazon-ssm-agent.service
else
  systemctl is-enabled amazon-ssm-agent.service
fi

# 6. Stop anything the package started and discard whatever it initialized
systemctl stop neo4j || true
rm -rf /var/lib/neo4j/data/*

# 7. Static neo4j.conf settings (UserData adds the advertised addresses and the memory settings)
set_neo4j_conf dbms.security.procedures.unrestricted 'apoc.meta.data'
set_neo4j_conf server.default_listen_address 0.0.0.0
set_neo4j_conf server.bolt.listen_address 0.0.0.0:7687
set_neo4j_conf server.http.listen_address 0.0.0.0:7474
set_neo4j_conf internal.dbms.cypher_ip_blocklist "10.0.0.0/8,100.64.0.0/10,172.16.0.0/12,192.168.0.0/16,169.254.169.0/24,fc00::/7,fe80::/10,ff00::/8"

# 8. APOC core and the GenAI plugin ship with the package; activate them (the bake fails if a jar isn't found)
cp /var/lib/neo4j/labs/apoc-*-core.jar /var/lib/neo4j/plugins/ || \
  cp /var/lib/neo4j/products/apoc-*-core.jar /var/lib/neo4j/plugins/
chown neo4j:neo4j /var/lib/neo4j/plugins/apoc-*-core.jar
for jar in /var/lib/neo4j/products/neo4j-genai-plugin-*.jar; do
  [ -e "$jar" ] || { echo "ERROR: GenAI plugin jar not found in /var/lib/neo4j/products"; exit 1; }
  cp "$jar" /var/lib/neo4j/plugins/
  chown neo4j:neo4j "/var/lib/neo4j/plugins/$(basename "$jar")"
done
# Only procedures and functions matching the allowlist load, so it must name APOC and both GenAI
# namespaces: genai.* (the deprecated functions) and ai.* (their replacements)
set_neo4j_conf dbms.security.procedures.allowlist 'apoc.*,genai.*,ai.*'
grep -n -E '^dbms\.security\.procedures\.' /etc/neo4j/neo4j.conf || true
# No server.jvm.additional edits: the package already ships the JDK vector API and native-access
# options as active lines. No restart either: Neo4j must not start before step 10 sets the
# initial password.

# 9. systemd: never start Neo4j unless the data volume is mounted; raise the open-file limit
mkdir -p /etc/systemd/system/neo4j.service.d
printf '[Unit]\nRequiresMountsFor=/var/lib/neo4j/data\n\n[Service]\nLimitNOFILE=60000\n' > /etc/systemd/system/neo4j.service.d/override.conf
systemctl daemon-reload

# 10. Smoke test Java + Neo4j + APOC with a throwaway password
neo4j-admin dbms set-initial-password 'bake-smoke-test-1'
chown -R neo4j:neo4j /var/lib/neo4j/data
systemctl start neo4j
for i in $(seq 1 60); do
  cypher-shell -a bolt://localhost:7687 -u neo4j -p 'bake-smoke-test-1' 'RETURN 1;' >/dev/null 2>&1 && break
  sleep 2
done
cypher-shell -a bolt://localhost:7687 -u neo4j -p 'bake-smoke-test-1' \
  'CALL dbms.components() YIELD name, versions, edition RETURN name, versions, edition;'
cypher-shell -a bolt://localhost:7687 -u neo4j -p 'bake-smoke-test-1' 'RETURN apoc.version() AS apoc;'

# 11. Leave no database, password, logs, keys or machine identity in the image;
#     neo4j.service stays disabled so it can't start before UserData mounts the data volume
systemctl stop neo4j
systemctl disable neo4j
rm -rf /var/lib/neo4j/data/* /var/log/neo4j/*
apt-get clean
rm -f /home/ubuntu/.ssh/authorized_keys /root/.bash_history /home/ubuntu/.bash_history
# Drop the builder's journal so the image carries none of this build's logs
journalctl --rotate
journalctl --vacuum-time=1s
cloud-init clean --logs --machine-id
echo BAKE_COMPLETE
EOF
bash -n /root/bake-neo4j-ce.sh && echo SYNTAX_OK
sha256sum /root/bake-neo4j-ce.sh
nohup bash /root/bake-neo4j-ce.sh > /root/bake-neo4j-ce.log 2>&1 & tail -f /root/bake-neo4j-ce.log
```

When the log ends with `BAKE_COMPLETE`, look back in it for:
- the `dbms.components()` row, showing `community` and your version
- the `apoc.version()` result

Then press Ctrl-C and run:

```bash
rm -f /root/bake-neo4j-ce.sh /root/bake-neo4j-ce.log
exit
exit
```

If the log stops at `apt-get install` with a version error, edit `NEO4J_VERSION` in the script to one printed by `apt-cache madison` just above the error, then run the `nohup` line again.

#### Step 8: Create the AMI (back on your Mac)

`create-image` reboots the builder so the snapshot is consistent.

```bash
AMI_NAME="cori-agent-neo4j-ce-2026.01.3-ubuntu-26.04-$(date +%Y%m%d%H%M)"
NEO4J_AMI_ID=$(aws ec2 create-image --instance-id "$BUILDER_ID" --name "$AMI_NAME" \
  --description "Neo4j CE 2026.01.3 + APOC on Ubuntu 26.04 (cori.agent)" \
  --tag-specifications "ResourceType=image,Tags=[{Key=Name,Value=$AMI_NAME}]" \
                       "ResourceType=snapshot,Tags=[{Key=Name,Value=$AMI_NAME}]" \
  --query ImageId --output text)
aws ec2 wait image-available --image-ids "$NEO4J_AMI_ID"
echo "$NEO4J_AMI_ID"

export NEO4J_PKG_VERSION=2026.09.0
export AMI_NAME="cori-agent-neo4j-ce-${NEO4J_PKG_VERSION}-ubuntu-26.04-$(date +%Y%m%d%H%M)"
export NEO4J_AMI_ID=$(aws ec2 create-image --instance-id "$BUILDER_ID" --name "$AMI_NAME" \
  --description "Neo4j CE ${NEO4J_PKG_VERSION} + APOC on Ubuntu 26.04 (cori.agent)" \
  --tag-specifications "ResourceType=image,Tags=[{Key=Name,Value=$AMI_NAME}]" "ResourceType=snapshot,Tags=[{Key=Name,Value=$AMI_NAME}]" --query ImageId --output text)
aws ec2 wait image-available --image-ids "$NEO4J_AMI_ID" && echo "$NEO4J_AMI_ID"
```

If the `wait` times out, run it again.

#### Step 9: Terminate the builder

```bash
aws ec2 terminate-instances --instance-ids "$BUILDER_ID"
```

#### Step 10: Put the AMI ID in the code, deploy your sandbox, and check the instance

Replace `ami-REPLACE_WITH_BAKED_NEO4J_CE_AMI` in `NEO4J_AMI_ID` with the ID from step 8 and deploy your sandbox. Then get `Neo4jInstanceId` from the kb nested stack's outputs and check it:

```bash
aws ssm start-session --target <Neo4jInstanceId>
sudo tail -n 30 /var/log/neo4j-userdata.log
systemctl status neo4j --no-pager
```

To open Neo4j Browser, forward both ports (two terminals), open http://localhost:7474, and connect to `bolt://localhost:7687`. The username is `neo4j` and the password is the secret named in `Neo4jPasswordSecretArn`.

```bash
aws ssm start-session --target <Neo4jInstanceId> --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["7474"],"localPortNumber":["7474"]}'
aws ssm start-session --target <Neo4jInstanceId> --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["7687"],"localPortNumber":["7687"]}'
```

#### Step 11: Upgrading Neo4j or patching the OS

The instance can't reach the internet, so upgrades and OS patches come only from rebaking: repeat steps 2 and 4–10 with a new `NEO4J_VERSION`. A new `NEO4J_AMI_ID` replaces the instance. That runs into the data-volume attachment problem from last time, so detach the volume (or delete the sandbox's Neo4j instance) before you deploy the new AMI.

The bake script is also saved at `/tmp/claude-0/-home-user/319bbda0-1503-5bb3-b7df-fd7c6f0b43cc/scratchpad/bake-neo4j-ce.sh`.

## neo4j-generative-ai-aws

The rest of this repo includes sample notebooks and a web application which shows how Amazon Bedrock and Titan can be used with Neo4j. Use `.env` to set the following environment variables:

```bash
NEO4J_URI=bolt://...:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=...
NEO4J_DATABASE=neo4j
```

We will explore how to leverage generative AI to build and consume a knowledge graph in Neo4j.

The dataset we're using is from the SEC's EDGAR system.  It was downloaded using [these scripts](https://github.com/neo4j-partners/neo4j-sec-edgar-form13).

The dataflow in this demo consists of two parts:
1. Ingestion - we read the EDGAR files with Bedrock, extracting entities and relationships from them which is then ingested into a Neo4j database deployed from [AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-akmzjikgawgn4).
2. Consumption - A user inputs natural language into a chat UI.  Bedrock converts that to Neo4j Cypher which is run against the database.  This flow allows non technical users to query the database.

## Setup Sagemaker Studio Environment
To get started setting up the demo, clone this repo into a [SageMaker Studio](https://aws.amazon.com/sagemaker/studio/) environment and then run through the notebooks numbered 1 and 2.

## Deploy Neo4j AuraDS Professional
This demo requires a Neo4j instance.  You can deploy that using the AWS Marketplace listing [here](https://aws.amazon.com/marketplace/pp/prodview-2t3o7mnw5ypee).

## Enable AWS IAM permissions for Bedrock
The AWS identity you assume from your notebook environment (which is the [*Studio/notebook Execution Role*](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html) from SageMaker, or could be a role or IAM User for self-managed notebooks), must have sufficient [AWS IAM permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html) to call the Amazon Bedrock service.

To grant Bedrock access to your identity, you can:

- Open the [AWS IAM Console](https://us-east-1.console.aws.amazon.com/iam/home?#)
- Find your [Role](https://us-east-1.console.aws.amazon.com/iamv2/home?#/roles) (if using SageMaker or otherwise assuming an IAM Role), or else [User](https://us-east-1.console.aws.amazon.com/iamv2/home?#/users)
- Select *Add Permissions > Create Inline Policy* to attach new inline permissions, open the *JSON* editor and paste in the below example policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "BedrockFullAccess",
            "Effect": "Allow",
            "Action": ["bedrock:*"],
            "Resource": "*"
        }
    ]
}
```

> ⚠️ **Note:** With Amazon SageMaker, your notebook execution role will typically be *separate* from the user or role that you log in to the AWS Console with. If you'd like to explore the AWS Console for Amazon Bedrock, you'll need to grant permissions to your Console user/role too.

For more information on the fine-grained action and resource permissions in Bedrock, check out the Bedrock Developer Guide.

## UI
The UI application is based on Streamlit. In this example we're going to show how to run it on an [AWS EC2 Instance (EC2)](https://console.aws.amazon.com/ec2/) VM.  First, deploy a VM. You can use [this guide to spin off an Amazon Linux VM](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)

We are going to use AWS CLI. You need to [follow these steps](https://docs.aws.amazon.com/cli/latest/userguide/cli-authentication-short-term.html) to configure the credentials to use the CLI option 

Login using AWS credentials via the `aws` cli.

    aws configure
        

Next, login to the new VM instance using AWS CLI:

    export INSTANCE_ID=<YOUR_EC2_INSTANCE_ID>
    aws ec2-instance-connect ssh --instance-id $INSTANCE_ID

We're going to be running the application on port 80.  That requires root access, so first:

    sudo su

Then you'll need to install git and clone this repo:

    yum install -y git
    mkdir -p /app
    cd /app
    git clone https://github.com/neo4j-partners/neo4j-generative-ai-aws.git
    cd neo4j-generative-ai-aws

Let's install python & pip first:

    yum install -y python
    yum install -y pip

Now, let's create a Virtual Environment to isolate our Python environment and activate it

    yum install -y virtualenv
    python3 -m venv /app/venv/genai
    source /app/venv/genai/bin/activate

To install Streamlit and other dependencies:

    cd ui
    pip install -r requirements.txt

Check if `streamlit` command is accessible from PATH by running this command:

    streamlit --version

If not, you need to add the `streamlit` binary to PATH variable like below:

    export PATH="/app/venv/genai/bin:$PATH"

Next up you'll need to create a secrets file for the app to use.  Open the file and edit it:

    cd streamlit
    cd .streamlit
    cp secrets.toml.example secrets.toml
    vi secrets.toml

You will now need to edit that file to reflect your credentials. The file has the following variables:

    SERVICE_NAME = "" #e.g. bedrock-runtime
    REGION_NAME = "" #e.g. us-west-2
    CYPHER_MODEL = "" #e.g. anthropic.claude-v2
    ACCESS_KEY = "AWS ACCESS KEY" #provide the access key with bedrock access
    SECRET_KEY = "AWS SECRET KEY" #provide the secret key with bedrock access
    NEO4J_HOST = "" #NEO4J_AURA_DS_URL
    NEO4J_PORT = "7687"
    NEO4J_USER = "neo4j"
    NEO4J_PASSWORD = "" #Neo4j password
    NEO4J_DB = "neo4j"

Now we can run the app with the commands:

    cd ..
    streamlit run Home.py --server.port=80

Optionally, you can run the app in another screen session to ensure the app continues to run even if you disconnect from the ec2 instance:

    screen -S run_app
    cd ..
    streamlit run Home.py --server.port=80    

You can use `Ctrl+a` `d` to exit the screen with the app still running and enter back into the screen with `screen -r`. To kill the screen session, use the command `screen -XS run_app quit`.

On the VM to run on port 80:
- Ensure you are a root or has access to run on port 80
- Ensure that the VM has port 80 open for HTTP access. You might need to open that port or any other via firewall rules as mentioned [here](https://repost.aws/knowledge-center/connect-http-https-ec2). 

Once deployed, you will be able to see the Dashboard and Chat UI:

## Dashboard
![Dashboard](images/dash.png)
## Chat UI
![Chat](images/chat.png)

From the Chat UI, you can ask questions like:
1. Which of the managers own Amazon?
2. How many managers own more than 100 companies and who are they?
3. Which manager own all the FAANG stocks?
