Below is a **fully complete**, **production-grade**, **copy-paste setup** for:    
**✅ EKS → Fluent Bit → CloudWatch (Hot)**  
**✅ CloudWatch → Firehose → S3 (Warm/Cold)**  
**✅ INCLUDING node logs (/var/log/messages, syslog, dmesg, kubelet, container runtime)**  

# Architecture  
```text
Node logs (/var/log/messages, syslog, dmesg, journald)
Pod logs (/var/log/containers)
        ↓
Fluent Bit (DaemonSet on every node)
        ↓
CloudWatch Logs (15 days retention)
        ↓
Subscription Filter
        ↓
Kinesis Firehose
        ↓
S3 (lifecycle → IA → Glacier → Deep Archive)
```
## STEP 1 — Create S3 Bucket  
```bash
aws s3api create-bucket \
  --bucket my-eks-logs-bucket \
  --region us-east-1
```

## STEP 2 — Add Lifecycle Policy   
| Stage      | Service         | Duration     |  
| ---------- | --------------- | ------------ |  
| 🔥 Hot     | CloudWatch      | 0–15 days    |  
| 📦 Warm    | S3 STANDARD     | 0–30 days    |  
| 💾 Cooler  | S3 STANDARD_IA  | 30–90 days  |  
| ❄️ Cold    | S3 GLACIER      | 90–180 days |  
| 🧊 Archive | S3 DEEP_ARCHIVE | 180+ days    |  


`lifecycle.json` 
```bash
cat <<EOF > lifecycle.json
{
  "Rules": [
    {
      "ID": "log-lifecycle",
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" },
        { "Days": 180, "StorageClass": "DEEP_ARCHIVE" }
      ]
    }
  ]
}
EOF
```
```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-eks-logs-bucket \
  --lifecycle-configuration file://lifecycle.json
```

## STEP 3 — IAM ROLE (IRSA for Fluent Bit)  
**Create Policy`fluent-bit-policy.json`**  
```bash
cat <<EOF > fluent-bit-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:PutLogEvents",
        "logs:CreateLogStream",
        "logs:CreateLogGroup",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```
```bash
aws iam create-policy \
  --policy-name FluentBitCloudWatchPolicy \
  --policy-document file://fluent-bit-policy.json
```

**Create IAM Role (IRSA)**  
```bash
eksctl create iamserviceaccount \
  --name fluent-bit \
  --namespace amazon-cloudwatch \
  --cluster <YOUR_CLUSTER_NAME> \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/FluentBitCloudWatchPolicy \
  --approve \
  --region us-east-1
```

## STEP 4 — Create Firehose (CloudWatch → S3)  
Create Policy `firehose.json`
```bash
cat <<EOF > firehose.json
{
  "RoleARN": "arn:aws:iam::<ACCOUNT_ID>:role/firehose-role",
  "BucketARN": "arn:aws:s3:::my-eks-logs-bucket",
  "Prefix": "logs/",
  "BufferingHints": {
    "SizeInMBs": 5,
    "IntervalInSeconds": 300
  },
  "CompressionFormat": "GZIP"
}
EOF
```

```bash
aws firehose create-delivery-stream \
  --delivery-stream-name eks-logs-firehose \
  --delivery-stream-type DirectPut \
  --s3-destination-configuration file://firehose.json
```

## STEP 5 — Install Fluent Bit (FULL CONFIG WITH NODE LOGS)  
```bash
helm repo add aws https://aws.github.io/eks-charts
helm repo update
```

## ✅ FINAL `values.yaml` (THIS IS THE IMPORTANT PART)  
```yaml
serviceAccount:
  create: false
  name: fluent-bit

cloudWatch:
  enabled: true
  region: us-east-1

logs:
  enabled: true

# =========================
# INPUTS
# =========================
input:
  tail:
    enabled: true
    path: /var/log/containers/*.log
    parser: docker
    tag: kube.*

  systemd:
    enabled: true
    tag: host.systemd.*
    systemdFilter:
      - _SYSTEMD_UNIT=kubelet.service
      - _SYSTEMD_UNIT=containerd.service
      - _SYSTEMD_UNIT=docker.service

# 🔥 CRITICAL — NODE FILE LOGS
extraInputs: |
  [INPUT]
      Name              tail
      Path              /var/log/messages
      Tag               host.messages
      Refresh_Interval  5
      Mem_Buf_Limit     50MB
      Skip_Long_Lines   On

  [INPUT]
      Name              tail
      Path              /var/log/syslog
      Tag               host.syslog
      Refresh_Interval  5
      Mem_Buf_Limit     50MB
      Skip_Long_Lines   On

  [INPUT]
      Name              tail
      Path              /var/log/dmesg
      Tag               host.dmesg
      Refresh_Interval  10
      Mem_Buf_Limit     10MB

# =========================
# FILTERS
# =========================
filters:
  kubernetes:
    enabled: true
    match: kube.*

# =========================
# OUTPUTS
# =========================
extraOutputs: |
  # POD LOGS
  [OUTPUT]
      Name cloudwatch_logs
      Match kube.*
      region us-east-1
      log_group_name /eks/pod-logs
      log_stream_prefix pod-
      auto_create_group true

  # NODE LOGS
  [OUTPUT]
      Name cloudwatch_logs
      Match host.*
      region us-east-1
      log_group_name /eks/node-logs
      log_stream_prefix node-
      auto_create_group true
```
**Install Fluent Bit**  
```bash
helm install aws-for-fluent-bit aws/aws-for-fluent-bit \
  -n amazon-cloudwatch \
  --create-namespace \
  -f values.yaml
```
# STEP 6 — CloudWatch → Firehose → S3  
```bash
aws logs put-subscription-filter \
  --log-group-name "/eks/node-logs" \
  --filter-name "node-to-s3" \
  --filter-pattern "" \
  --destination-arn arn:aws:firehose:us-east-1:<ACCOUNT_ID>:deliverystream/eks-logs-firehose
```
```bash
aws logs put-subscription-filter \
  --log-group-name "/eks/pod-logs" \
  --filter-name "pod-to-s3" \
  --filter-pattern "" \
  --destination-arn arn:aws:firehose:us-east-1:<ACCOUNT_ID>:deliverystream/eks-logs-firehose
```

# STEP 7 — Set CloudWatch Retention (HOT = 15 DAYS)  
```bash
aws logs put-retention-policy \
  --log-group-name "/eks/node-logs" \
  --retention-in-days 15

aws logs put-retention-policy \
  --log-group-name "/eks/pod-logs" \
  --retention-in-days 15
```

## STEP 8 — VERIFY   
```bash
kubectl get pods -n amazon-cloudwatch
```
```bash
kubectl logs -n amazon-cloudwatch <fluent-bit-pod>
```

## 📊 FINAL LOG COVERAGE  
**✅ NODE LOGS (NOW INCLUDED)**    
 - /var/log/messages ✅  
 - /var/log/syslog ✅  
 - /var/log/dmesg ✅  
 - kubelet logs ✅  
 - container runtime logs ✅

**✅ POD LOGS**
 - /var/log/containers/*.log

## 📦 FINAL STORAGE STRATEGY  
| Stage      | Service         | Duration     |  
| ---------- | --------------- | ------------ |  
| 🔥 Hot     | CloudWatch      | 0–15 days    |  
| 📦 Warm    | S3 STANDARD     | 0–30 days    |  
| 💾 Cooler  | S3 STANDARD_IA  | 30–90 days  |  
| ❄️ Cold    | S3 GLACIER      | 90–180 days |  
| 🧊 Archive | S3 DEEP_ARCHIVE | 180+ days    |  

## ⚖️ Balance Between Cost vs Access  
| Stage        | Access Speed | Cost        | Usage             |  
| ------------ | ------------ | ----------- | ----------------- |  
| CloudWatch   | Instant      | 💸 High     | Debugging         |  
| S3 Standard  | Fast         | 💰 Medium   | Recent logs       |  
| Standard IA  | Medium       | 💵 Low      | Occasional access |  
| Glacier      | Slow         | 🪙 Very low | Rare              |  
| Deep Archive | Very slow    | 🧊 Cheapest | Compliance        |  


## 🚨 ZERO-MISS CHECKLIST  
✔ IRSA configured  
✔ Fluent Bit DaemonSet running  
✔ `/var/log/messages` explicitly added  
✔ systemd logs included  
✔ CloudWatch log groups separated  
✔ Firehose connected  
✔ S3 lifecycle enabled    

