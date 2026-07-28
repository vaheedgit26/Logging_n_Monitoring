# ARCHITECTURE 
```text
EKS (Pods + Nodes)
        ↓
Fluent Bit (DaemonSet)
        ↓
CloudWatch Logs (15 days retention)
        ↓
Subscription Filter
        ↓
Firehose (GZIP + buffering)
        ↓
S3 (lifecycle rules)
```
**Uses:**  
 - Amazon CloudWatch  
 - Amazon Kinesis Data Firehose  
 - Amazon S3

## ✅ STEP 1 — Create S3 Bucket  
```bash
aws s3 mb s3://my-eks-logs-bucket
```
## ✅ STEP 2 — S3 Lifecycle Policy (YOUR REQUIREMENT)
⚠️ Note: First 30 days = STANDARD (default)  
```bash
cat <<EOF > lifecycle.json
{
  "Rules": [
    {
      "ID": "log-lifecycle",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        },
        {
          "Days": 180,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ]
    }
  ]
}
EOF
```
Apply:
```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-eks-logs-bucket \
  --lifecycle-configuration file://lifecycle.json
```
## ✅ STEP 3 — IAM Role for Firehose  
```bash
cat <<EOF > firehose-trust.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "firehose.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```
```bash
aws iam create-role \
  --role-name firehose-role \
  --assume-role-policy-document file://firehose-trust.json
```
**Attach Policy:**  
```bash
cat <<EOF > firehose-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:AbortMultipartUpload",
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-eks-logs-bucket",
        "arn:aws:s3:::my-eks-logs-bucket/*"
      ]
    }
  ]
}
EOF
```
```bash
aws iam put-role-policy \
  --role-name firehose-role \
  --policy-name firehose-s3-policy \
  --policy-document file://firehose-policy.json
```
## ✅ STEP 4 — Create Firehose  
```bash
cat <<EOF > firehose.json
{
  "DeliveryStreamName": "eks-logs-firehose",
  "DeliveryStreamType": "DirectPut",
  "ExtendedS3DestinationConfiguration": {
    "RoleARN": "arn:aws:iam::<ACCOUNT_ID>:role/firehose-role",
    "BucketARN": "arn:aws:s3:::my-eks-logs-bucket",
    "Prefix": "logs/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
    "ErrorOutputPrefix": "errors/",
    "BufferingHints": {
      "SizeInMBs": 5,
      "IntervalInSeconds": 300
    },
    "CompressionFormat": "GZIP"
  }
}
EOF
```
**Create:**  
```bash
aws firehose create-delivery-stream --cli-input-json file://firehose.json
```

## ✅ STEP 5 — CloudWatch Log Groups (15 days HOT)  
```bash
aws logs create-log-group --log-group-name "/eks/pod-logs"
aws logs create-log-group --log-group-name "/eks/node-logs"
```
**Set retention:**  
```bash
aws logs put-retention-policy \
  --log-group-name "/eks/pod-logs" \
  --retention-in-days 15

aws logs put-retention-policy \
  --log-group-name "/eks/node-logs" \
  --retention-in-days 15
```

## ✅ STEP 6 — IAM Role (CloudWatch → Firehose)  
```bash
cat <<EOF > cw-trust.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "logs.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```
```bash
aws iam create-role \
  --role-name cw-to-firehose-role \
  --assume-role-policy-document file://cw-trust.json
````
**Policy:**  
```bash
cat <<EOF > cw-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "firehose:PutRecord",
        "firehose:PutRecordBatch"
      ],
      "Resource": "arn:aws:firehose:us-east-1:<ACCOUNT_ID>:deliverystream/eks-logs-firehose"
    }
  ]
}
EOF
```
```bash
aws iam put-role-policy \
  --role-name cw-to-firehose-role \
  --policy-name cw-firehose \
  --policy-document file://cw-policy.json
```

## ✅ STEP 7 — Subscription Filter (CloudWatch → Firehose)  
```bash
aws logs put-subscription-filter \
  --log-group-name "/eks/pod-logs" \
  --filter-name "pods-to-s3" \
  --filter-pattern "" \
  --destination-arn arn:aws:firehose:us-east-1:<ACCOUNT_ID>:deliverystream/eks-logs-firehose \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/cw-to-firehose-role
```
```bash
aws logs put-subscription-filter \
  --log-group-name "/eks/node-logs" \
  --filter-name "nodes-to-s3" \
  --filter-pattern "" \
  --destination-arn arn:aws:firehose:us-east-1:<ACCOUNT_ID>:deliverystream/eks-logs-firehose \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/cw-to-firehose-role
```

## ✅ STEP 8 — Install Fluent Bit (EKS)  
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```
**values.yaml (IMPORTANT — includes node logs)**  
```yaml
# ======================================================
# RBAC
# Required for Kubernetes metadata enrichment
# ======================================================

rbac:
  create: true


# ======================================================
# Service Account
# ======================================================
# For production add IRSA annotation
# ======================================================

serviceAccount:
  create: true
  name: aws-for-fluent-bit

  # Uncomment after creating IAM Role for Fluent Bit
  #
  # annotations:
  #   eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/fluent-bit-cloudwatch-role



# ======================================================
# Run Fluent Bit on every node
# ======================================================

tolerations:
  - operator: Exists


# Required for node/systemd log collection

hostNetwork: true

dnsPolicy: ClusterFirstWithHostNet



# ======================================================
# Fluent Bit Configuration
# ======================================================

config:

  # ----------------------------------------------------
  # SERVICE
  # ----------------------------------------------------

  service: |

    [SERVICE]
        Flush                     5
        Log_Level                 info
        Daemon                    Off
        Parsers_File              parsers.conf

        HTTP_Server               On
        HTTP_Listen               0.0.0.0
        HTTP_PORT                 2020



  # ====================================================
  # INPUTS
  # ====================================================

  inputs: |


    # --------------------------------------------------
    # 1. POD LOGS
    # --------------------------------------------------
    #
    # Source:
    # /var/log/containers/*.log
    #
    # Kubernetes application stdout/stderr logs
    #
    # EKS uses containerd, therefore CRI parser
    #

    [INPUT]

        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            cri
        DB                /var/lib/fluent-bit/flb_kube.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    # --------------------------------------------------
    # 2. NODE OS LOGS
    # --------------------------------------------------
    #
    # Amazon Linux:
    #
    # /var/log/messages
    #
    # Contains:
    # kernel
    # OS events
    # system messages
    #

    [INPUT]
        Name              tail
        Tag               node.messages
        Path              /var/log/messages
        DB                /var/lib/fluent-bit/flb_node.db
        Mem_Buf_Limit     20MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    # --------------------------------------------------
    # 3. SYSTEMD JOURNAL LOGS
    # --------------------------------------------------
    #
    # Collect:
    #
    # kubelet
    # containerd
    #
    # from:
    #
    # /run/log/journal
    #

    [INPUT]
        Name              systemd
        Tag               node.systemd
        DB                /var/lib/fluent-bit/flb_systemd.db
        Systemd_Filter    _SYSTEMD_UNIT=kubelet.service
        Systemd_Filter    _SYSTEMD_UNIT=containerd.service
        Read_From_Tail    On

  # ====================================================
  # FILTERS
  # ====================================================

  filters: |

    # --------------------------------------------------
    # Kubernetes Metadata
    # --------------------------------------------------
    #
    # Adds:
    #
    # namespace
    # pod name
    # container name
    # labels
    #

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Merge_Log           On
        Keep_Log            Off
        Labels              On
        Annotations         Off

  # ====================================================
  # OUTPUTS
  # ====================================================

  outputs: |

    # --------------------------------------------------
    # POD LOGS
    # Fluent Bit --> CloudWatch
    # --------------------------------------------------

    [OUTPUT]
        Name                cloudwatch_logs
        Match               kube.*
        region              us-east-1
        log_group_name      /eks/pod-logs
        log_stream_prefix   pod-
        auto_create_group   true

    # --------------------------------------------------
    # NODE /var/log/messages
    # Fluent Bit --> CloudWatch
    # --------------------------------------------------

    [OUTPUT]
        Name                cloudwatch_logs
        Match               node.messages
        region              us-east-1
        log_group_name      /eks/node-logs
        log_stream_prefix   node-
        auto_create_group   true

    # --------------------------------------------------
    # SYSTEMD LOGS
    # Fluent Bit --> CloudWatch
    # --------------------------------------------------

    [OUTPUT]
        Name                cloudwatch_logs
        Match               node.systemd
        region              us-east-1
        log_group_name      /eks/node-systemd-logs
        log_stream_prefix   systemd-
        auto_create_group   true

# ======================================================
# Host Volumes
# ======================================================

daemonSetVolumes:
  # --------------------------------------
  # Node log directory
  # --------------------------------------

  - name: varlog
    hostPath:
      path: /var/log
      type: Directory

  # --------------------------------------
  # systemd journal
  # --------------------------------------

  - name: systemd
    hostPath:
      path: /run/log/journal
      type: Directory

  # --------------------------------------
  # Node identity
  # --------------------------------------

  - name: machine-id
    hostPath:
      path: /etc/machine-id
      type: File

  # --------------------------------------
  # Fluent Bit state database
  # --------------------------------------

  - name: fluent-bit-state
    emptyDir: {}

# ======================================================
# Volume Mounts inside Fluent Bit container
# ======================================================

daemonSetVolumeMounts:


  # Node logs
  - name: varlog
    mountPath: /var/log
    readOnly: true

  # systemd journal
  - name: systemd
    mountPath: /run/log/journal
    readOnly: true

  # node identity
  - name: machine-id
    mountPath: /etc/machine-id
    readOnly: true

  # Fluent Bit DB files
  - name: fluent-bit-state
    mountPath: /var/lib/fluent-bit
```
**Install:**  
```bash
helm upgrade --install aws-for-fluent-bit eks/aws-for-fluent-bit \
  -n amazon-cloudwatch \
  --create-namespace \
  -f values.yaml
```
**verify**
```bash
kubectl get pods -n amazon-cloudwatch
```

## ✅ FINAL RESULT  
**Logs Flow**:  
 - Pod logs → /eks/pod-logs  
 - Node logs (/var/log/messages) → /eks/node-logs  
 - Stored in CloudWatch for 15 days  
 - Streamed to Firehose  
 - Stored in S3 as .gz  
 - Lifecycle:

| Time      | Storage      |  
| --------- | ------------ |  
| 0–30 days | STANDARD     |  
| 30+       | STANDARD_IA  |  
| 90+       | GLACIER      |  
| 180+      | DEEP_ARCHIVE |  

## ✅ YOU NOW HAVE  
✔ Pod logs  
✔ Node logs  
✔ CloudWatch (hot 15 days)  
✔ S3 archival  
✔ Cost optimisation  
✔ Compression (GZIP)  
