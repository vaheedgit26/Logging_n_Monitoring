## ✅ STEP 1 — IRSA setup for Fluent Bit
**Create IAM Policy for Fluent Bit  `fluent-bit-cloudwatch-policy.json`**  
```bash
cat <<EOF > fluent-bit-cloudwatch-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```
**Create policy:**  
```bash
aws iam create-policy \
--policy-name fluent-bit-cloudwatch-policy \
--policy-document file://fluent-bit-cloudwatch-policy.json
```
**You will get:** 
```bash
arn:aws:iam::<ACCOUNT_ID>:policy/fluent-bit-cloudwatch-policy
```
**Create IAM Role for IRSA**  
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:amazon-cloudwatch:aws-for-fluent-bit"
        }
      }
    }
  ]
}
```
**Important part:**  
```json
"sub": "system:serviceaccount:amazon-cloudwatch:aws-for-fluent-bit"
```
**This must match:**  
```yaml
serviceAccount:
  name: aws-for-fluent-bit
```
and your namespace


**Attach Policy to Role**  
```bash
aws iam attach-role-policy \
--role-name fluent-bit-cloudwatch-role \
--policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/fluent-bit-cloudwatch-policy
```
**Update your values.yaml**  
```yaml
serviceAccount:
  create: true
  name: aws-for-fluent-bit
```
**to**  
```yaml
serviceAccount:
  create: true
  name: aws-for-fluent-bit

  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/fluent-bit-cloudwatch-role
```
**Now Fluent Bit gets AWS credentials automatically.**

## ✅ STEP 2 — CloudWatch Log Groups (15 days HOT)
**Create Log-groups**   
```bash
aws logs create-log-group --log-group-name "/eks/pod-logs"
aws logs create-log-group --log-group-name "/eks/node-logs"
aws logs create-log-group --log-group-name "/eks/node-systemd-logs"
```
**Set retention:**   
```bash
aws logs put-retention-policy \
  --log-group-name "/eks/pod-logs" \
  --retention-in-days 15

aws logs put-retention-policy \
  --log-group-name "/eks/node-logs" \
  --retention-in-days 15

aws logs put-retention-policy \
  --log-group-name "/eks/node-systemd-logs" \
  --retention-in-days 15
```

## ✅ STEP 3 — Install Fluent Bit (EKS)  
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```
**`values.yaml` (IMPORTANT — includes node logs)**  
```yaml
input:
  enabled: false

cloudWatchLogs:
  enabled: false

# hostNetwork: false
# dnsPolicy: ClusterFirst

# ======================================================
# RBAC (needed for Kubernetes metadata)
# ======================================================

rbac:
  create: true

# ======================================================
# Service Account (use IRSA in production)
# ======================================================

serviceAccount:
  create: true
  name: aws-for-fluent-bit

  # Uncomment after creating IAM Role for Fluent Bit
  #
  # annotations:
  #   eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/fluent-bit-cloudwatch-role

# ======================================================
# Scheduling (run Fluent Bit on all nodes)
# ======================================================
tolerations:
  - operator: Exists

# Required for node/systemd log collection
hostNetwork: true
dnsPolicy: ClusterFirstWithHostNet

# ======================================================
# Resources
# ======================================================
resources:
  limits:
    memory: 200Mi
    cpu: 200m
  requests:
    memory: 100Mi
    cpu: 100m

# ======================================================
# Fluent Bit Configuration
# ======================================================

config:

  # ====================================================
  # SERVICE
  # ====================================================

  service: |

    [SERVICE]
        Flush                     5
        Log_Level                 info
        Daemon                    Off
        Parsers_File              parsers.conf

        HTTP_Server               On
        HTTP_Listen               0.0.0.0
        HTTP_PORT                 2020

       storage.path              /var/lib/fluent-bit/storage
       storage.sync              normal
       storage.checksum          on
       storage.backlog.mem_limit 50M
       storage.total_limit_size  1G

  # ====================================================
  # INPUTS
  # ====================================================

  inputs: |

    # --------------------------------------------------
    # 1. POD LOGS
    # --------------------------------------------------
    # Source: /var/log/containers/*.log
    # Kubernetes application stdout/stderr logs
    # EKS uses containerd, therefore CRI parser

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            cri
        multiline.parser  docker, cri
        DB                /var/lib/fluent-bit/flb_kube.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    # --------------------------------------------------
    # 2. NODE OS LOGS
    # --------------------------------------------------
    # Amazon Linux: /var/log/messages
    # Contains: kernel, OS events, system messages
    
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
    # Collect: kubelet, containerd
    # from: /run/log/journal
   
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
    # Adds: namespace, pod name, container name and labels
    
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off
        Labels              On
        Annotations         Off

  # ====================================================
  # OUTPUTS
  # ====================================================

  outputs: |

    # --------------------------------------------------
    # POD LOGS (Fluent Bit --> CloudWatch)
    # --------------------------------------------------

    [OUTPUT]
        Name                cloudwatch_logs
        Match               kube.*
        region              us-east-1
        log_group_name      /eks/pod-logs
        log_stream_prefix   pod-${HOSTNAME}-
        auto_create_group   true

    # -------------------------------------------------------
    # NODE LOGS /var/log/messages (Fluent Bit --> CloudWatch)
    # -------------------------------------------------------

    [OUTPUT]
        Name                cloudwatch_logs
        Match               node.messages
        region              us-east-1
        log_group_name      /eks/node-logs
        log_stream_prefix   node-${HOSTNAME}-
        auto_create_group   true

    # -------------------------------------------------------------
    # SYSTEMD LOGS (kubelet + containerd) Fluent Bit --> CloudWatch
    # -------------------------------------------------------------

    [OUTPUT]
        Name                cloudwatch_logs
        Match               node.systemd
        region              us-east-1
        log_group_name      /eks/node-systemd-logs
        log_stream_prefix   systemd-${HOSTNAME}-
        auto_create_group   true

    # -------------------------------------------------------------
    # ALL LOGS (Fluent Bit --> KINESIS FIREHOSE)
    # -------------------------------------------------------------

    #[OUTPUT]
    # Name            kinesis_firehose
    # Match           *
    # region          ap-south-1
    # delivery_stream fluentbit-logs
    # Retry_Limit     False

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
