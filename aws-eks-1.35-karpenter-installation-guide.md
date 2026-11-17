# การติดตั้ง Karpenter บน AWS EKS 1.35

## คำแนะนำสำหรับ AWS EKS 1.35

จาก Compatibility Matrix ปัจจุบัน **Amazon EKS / Kubernetes 1.35 ต้องใช้ Karpenter ตั้งแต่ `v1.9.0` ขึ้นไป**

สำหรับ Production แนะนำ:

| Component | Version แนะนำ | เหตุผล |
|---|---:|---|
| `karpenter-crd` | **1.14.3** | เป็น LTS รองรับถึงกรกฎาคม 2027 |
| `karpenter` | **1.14.3** | ต้องใช้ version เดียวกับ CRD |
| Kubernetes / EKS | **1.35** | รองรับโดย Karpenter ตั้งแต่ 1.9 ขึ้นไป |

Karpenter `v1.14.3` เป็น LTS release และรองรับถึงเดือนกรกฎาคม 2027 จึงเหมาะกับ Production มากกว่า regular release ที่ต้องอัปเกรดบ่อยกว่า

> **ทางเลือก:** ปัจจุบัน Latest คือ `v1.15.2` ซึ่งรองรับ EKS 1.35 เช่นกัน แต่เป็น regular release ไม่ใช่ LTS หากระบบเน้นเสถียรภาพ ให้ใช้ `1.14.3`; หากต้องการ feature ล่าสุดและมีรอบอัปเกรดสม่ำเสมอ จึงค่อยเลือก `1.15.2`

ด้านล่างใช้ **`1.14.3`** เป็น baseline สำหรับ Production

---

## ภาพรวมสิ่งที่ต้องเตรียม

Karpenter ต้องมีองค์ประกอบหลักดังนี้:

1. **Karpenter Controller**
2. **Karpenter CRDs**
   - `NodePool`
   - `NodeClaim`
   - `EC2NodeClass`
3. **IAM Role สำหรับ Karpenter Controller**
4. **IAM Role สำหรับ EC2 Node**
5. **EKS Access Entry สำหรับ Node Role**
6. **SQS Queue สำหรับ Spot interruption**
7. **EventBridge Rules**
8. **Subnet และ Security Group tags**
9. **Managed Node Group ขนาดเล็กสำหรับรัน Karpenter Controller**
10. `EC2NodeClass` และ `NodePool`

AWS แนะนำให้มี IAM controller role, node role/instance profile, EKS access entry และ SQS interruption queue ก่อนติดตั้ง Helm chart

---

## สถาปัตยกรรมที่แนะนำ

```text
EKS Control Plane 1.35
        |
        +-- Managed Node Group: system
        |      |
        |      +-- Karpenter Controller
        |      +-- CoreDNS
        |      +-- VPC CNI
        |      +-- kube-proxy
        |
        +-- Karpenter NodePool
               |
               +-- NodeClaim
               +-- EC2 Instance
                    |
                    +-- Application Pods
```

### ข้อสำคัญ

**อย่าวาง Karpenter Controller ให้ทำงานเฉพาะบน node ที่ Karpenter สร้างเอง**

ควรมี Managed Node Group อย่างน้อย:

- 2 nodes
- กระจาย 2 Availability Zones
- instance เช่น `m6i.large`, `m7i.large` หรือ `m7g.large`
- ติด label เช่น `karpenter.sh/controller=true`
- Controller replicas อย่างน้อย 2

หาก Karpenter ลบ node ของตัวเอง แล้ว controller อยู่บน node นั้นเพียงจุดเดียว ระบบอาจไม่สามารถ provision node ใหม่ได้

---

## ขั้นตอนที่ 1: ตรวจสอบ EKS Cluster

ตัวอย่างใช้ Bash, AWS CloudShell หรือ Linux shell:

```bash
export CLUSTER_NAME="my-eks-cluster"
export AWS_REGION="ap-southeast-1"
export KARPENTER_NAMESPACE="karpenter"
export KARPENTER_VERSION="1.14.3"

export AWS_ACCOUNT_ID="$(
  aws sts get-caller-identity \
    --query Account \
    --output text
)"
```

ตรวจสอบ identity:

```bash
aws sts get-caller-identity
```

ตรวจสอบ EKS version:

```bash
aws eks describe-cluster \
  --name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --query 'cluster.version' \
  --output text
```

ผลลัพธ์ต้องเป็น:

```text
1.35
```

ตั้งค่า `kubectl`:

```bash
aws eks update-kubeconfig \
  --name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}"
```

ตรวจสอบ:

```bash
kubectl version
kubectl get nodes -o wide
kubectl get pods -A
```

---

## ขั้นตอนที่ 2: ตรวจสอบ Add-on ก่อนติดตั้ง

```bash
aws eks list-addons \
  --cluster-name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}"
```

ตรวจสอบ version:

```bash
aws eks describe-addon \
  --cluster-name "${CLUSTER_NAME}" \
  --addon-name vpc-cni \
  --region "${AWS_REGION}" \
  --query 'addon.addonVersion' \
  --output text

aws eks describe-addon \
  --cluster-name "${CLUSTER_NAME}" \
  --addon-name kube-proxy \
  --region "${AWS_REGION}" \
  --query 'addon.addonVersion' \
  --output text

aws eks describe-addon \
  --cluster-name "${CLUSTER_NAME}" \
  --addon-name coredns \
  --region "${AWS_REGION}" \
  --query 'addon.addonVersion' \
  --output text
```

ควรตรวจสอบให้แน่ใจว่า:

- VPC CNI รองรับ Kubernetes 1.35
- CoreDNS ทำงานปกติ
- kube-proxy ทำงานปกติ
- Nodes มี IP เพียงพอ
- Pod Security หรือ Admission Policy ไม่บล็อก namespace ของ Karpenter

Karpenter อาจสร้าง node ได้สำเร็จแต่ node ไม่พร้อมใช้งาน ถ้า VPC CNI เก่า, subnet ไม่มี IP หรือ CNI ไม่สามารถ allocate IP ให้ Pod ได้

---

## ขั้นตอนที่ 3: Tag Subnet และ Security Group

### 3.1 Tag Subnet

```bash
aws eks describe-cluster \
  --name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --query 'cluster.resourcesVpcConfig.subnetIds' \
  --output text
```

ใส่ tag ให้ subnet ที่ต้องการให้ Karpenter ใช้งาน:

```bash
aws ec2 create-tags \
  --resources subnet-xxxxxxxx subnet-yyyyyyyy \
  --region "${AWS_REGION}" \
  --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}"
```

แนะนำให้เลือก **Private Subnets อย่างน้อย 2 AZ**

```bash
aws ec2 describe-subnets \
  --subnet-ids subnet-xxxxxxxx subnet-yyyyyyyy \
  --region "${AWS_REGION}" \
  --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,AvailableIPs:AvailableIpAddressCount}' \
  --output table
```

### 3.2 Tag Security Group

```bash
export CLUSTER_SECURITY_GROUP="$(
  aws eks describe-cluster \
    --name "${CLUSTER_NAME}" \
    --region "${AWS_REGION}" \
    --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' \
    --output text
)"

aws ec2 create-tags \
  --resources "${CLUSTER_SECURITY_GROUP}" \
  --region "${AWS_REGION}" \
  --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}"
```

ตรวจสอบ:

```bash
aws ec2 describe-tags \
  --region "${AWS_REGION}" \
  --filters \
    "Name=key,Values=karpenter.sh/discovery" \
    "Name=value,Values=${CLUSTER_NAME}" \
  --output table
```

---

## ขั้นตอนที่ 4: สร้าง IAM Node Role

```bash
export KARPENTER_NODE_ROLE="KarpenterNodeRole-${CLUSTER_NAME}"
```

สร้าง trust policy:

```bash
cat > node-role-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name "${KARPENTER_NODE_ROLE}" \
  --assume-role-policy-document file://node-role-trust-policy.json
```

Attach policies:

```bash
aws iam attach-role-policy --role-name "${KARPENTER_NODE_ROLE}" --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name "${KARPENTER_NODE_ROLE}" --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --role-name "${KARPENTER_NODE_ROLE}" --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam attach-role-policy --role-name "${KARPENTER_NODE_ROLE}" --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

> ในสภาพแวดล้อมที่ใช้ IRSA หรือ Pod Identity สำหรับ `aws-node` อยู่แล้ว อาจไม่จำเป็นต้องใส่ `AmazonEKS_CNI_Policy` ที่ node role แต่การออกแบบสิทธิ์ต้องสอดคล้องกับวิธีที่ VPC CNI รับ credentials

---

## ขั้นตอนที่ 5: สร้าง EKS Access Entry

```bash
aws eks create-access-entry \
  --cluster-name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --principal-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${KARPENTER_NODE_ROLE}" \
  --type EC2_LINUX
```

ตรวจสอบ:

```bash
aws eks describe-access-entry \
  --cluster-name "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --principal-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${KARPENTER_NODE_ROLE}"
```

`EC2_LINUX` access entry มีไว้สำหรับ EC2 node และปกติไม่ต้อง associate `AmazonEKSClusterAdminPolicy`

---

## ขั้นตอนที่ 6: สร้าง SQS Interruption Queue

```bash
export INTERRUPTION_QUEUE="${CLUSTER_NAME}-karpenter"

aws sqs create-queue \
  --queue-name "${INTERRUPTION_QUEUE}" \
  --region "${AWS_REGION}" \
  --attributes '{"MessageRetentionPeriod":"300","SqsManagedSseEnabled":"true"}'
```

```bash
export QUEUE_URL="$(aws sqs get-queue-url --queue-name "${INTERRUPTION_QUEUE}" --region "${AWS_REGION}" --query QueueUrl --output text)"
export QUEUE_ARN="$(aws sqs get-queue-attributes --queue-url "${QUEUE_URL}" --region "${AWS_REGION}" --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)"
```

ต้องสร้าง EventBridge rules สำหรับเหตุการณ์อย่างน้อย:

- EC2 Spot Instance Interruption Warning
- EC2 Instance Rebalance Recommendation
- EC2 Instance State-change Notification
- AWS Health Scheduled Change

ใน Production แนะนำให้สร้าง IAM, SQS และ EventBridge ด้วย **Terraform หรือ CloudFormation official Karpenter template** แทนการสร้างทีละคำสั่ง เพื่อให้ queue policy และ event targets ครบถ้วนและตรวจสอบ drift ได้ง่าย

---

## ขั้นตอนที่ 7: สร้าง IAM Role สำหรับ Karpenter Controller

Karpenter controller ต้องมีสิทธิ์สำคัญ เช่น:

- `ec2:CreateFleet`
- `ec2:RunInstances`
- `ec2:CreateLaunchTemplate`
- `ec2:DeleteLaunchTemplate`
- `ec2:TerminateInstances`
- `ec2:Describe*`
- `pricing:GetProducts`
- `ssm:GetParameter`
- `iam:PassRole`
- `eks:DescribeCluster`
- `sqs:ReceiveMessage`
- `sqs:DeleteMessage`
- `sqs:GetQueueUrl`

ควรจำกัด `iam:PassRole` เฉพาะ Node Role และห้ามใช้ `Resource: "*"`

ตัวอย่าง IRSA:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --approve

eksctl create iamserviceaccount \
  --cluster "${CLUSTER_NAME}" \
  --region "${AWS_REGION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --name karpenter \
  --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
  --attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}" \
  --approve \
  --role-only

export KARPENTER_CONTROLLER_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterControllerRole-${CLUSTER_NAME}"
```

---

## ขั้นตอนที่ 8: ติดตั้ง `karpenter-crd`

```bash
aws ecr-public get-login-password \
  --region us-east-1 |
helm registry login \
  --username AWS \
  --password-stdin public.ecr.aws

kubectl create namespace "${KARPENTER_NAMESPACE}" \
  --dry-run=client -o yaml |
kubectl apply -f -

helm upgrade --install karpenter-crd \
  oci://public.ecr.aws/karpenter/karpenter-crd \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --wait \
  --timeout 10m
```

ตรวจสอบ:

```bash
helm list -n "${KARPENTER_NAMESPACE}"
kubectl get crd | grep -E 'karpenter.sh|karpenter.k8s.aws'
```

ควรพบ:

```text
nodepools.karpenter.sh
nodeclaims.karpenter.sh
ec2nodeclasses.karpenter.k8s.aws
```

---

## ขั้นตอนที่ 9: ติดตั้ง Karpenter Controller

สร้าง `karpenter-values.yaml`:

```yaml
serviceAccount:
  create: true
  name: karpenter
  annotations:
    eks.amazonaws.com/role-arn: REPLACE_CONTROLLER_ROLE_ARN

settings:
  clusterName: REPLACE_CLUSTER_NAME
  interruptionQueue: REPLACE_INTERRUPTION_QUEUE

replicas: 2

controller:
  resources:
    requests:
      cpu: 1
      memory: 1Gi
    limits:
      memory: 1Gi

podDisruptionBudget:
  name: karpenter
  maxUnavailable: 1

logLevel: info
```

ติดตั้ง:

```bash
helm upgrade --install karpenter \
  oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --values karpenter-values.yaml \
  --wait \
  --timeout 15m \
  --atomic
```

ตรวจสอบ:

```bash
kubectl get deployment,pod,serviceaccount -n "${KARPENTER_NAMESPACE}"
kubectl rollout status deployment/karpenter -n "${KARPENTER_NAMESPACE}" --timeout=5m
kubectl logs -n "${KARPENTER_NAMESPACE}" deployment/karpenter --all-containers=true --tail=200
```

---

## ขั้นตอนที่ 10: สร้าง EC2NodeClass

ควร pin AMI ใน Production:

```bash
aws ssm get-parameter \
  --region "${AWS_REGION}" \
  --name "/aws/service/eks/optimized-ami/1.35/amazon-linux-2023/x86_64/standard/recommended/image_id" \
  --query 'Parameter.Value' \
  --output text
```

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  role: KarpenterNodeRole-REPLACE_CLUSTER_NAME
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: REPLACE_CLUSTER_NAME
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: REPLACE_CLUSTER_NAME
  amiSelectorTerms:
    - alias: al2023@REPLACE_PINNED_RELEASE
  metadataOptions:
    httpEndpoint: enabled
    httpProtocolIPv6: disabled
    httpPutResponseHopLimit: 1
    httpTokens: required
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        encrypted: true
        deleteOnTermination: true
```

> อย่า copy ค่า release ตัวอย่างไปใช้ทันที ให้ดึงค่าจริงและทดสอบใน Non-Production ก่อน

```bash
kubectl apply -f ec2nodeclass.yaml
kubectl get ec2nodeclass
kubectl describe ec2nodeclass default
```

---

## ขั้นตอนที่ 11: สร้าง NodePool

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        workload-type: application
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
        - key: kubernetes.io/os
          operator: In
          values: [linux]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: [c, m, r]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["5"]
      expireAfter: 720h
      terminationGracePeriod: 24h
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 5m
    budgets:
      - nodes: "10%"
  limits:
    cpu: "500"
    memory: 1000Gi
  weight: 10
```

```bash
kubectl apply -f nodepool.yaml
kubectl get nodepool
kubectl describe nodepool default
```

---

## ขั้นตอนที่ 12: ทดสอบ Provisioning

```bash
kubectl create deployment inflate --image=public.ecr.aws/eks-distro/kubernetes/pause:3.10
kubectl scale deployment inflate --replicas=5
kubectl get pods -w
kubectl get nodeclaims -w
kubectl get nodes -L karpenter.sh/nodepool,karpenter.sh/capacity-type
kubectl logs -n "${KARPENTER_NAMESPACE}" deployment/karpenter --all-containers=true --follow
```

หลังทดสอบ:

```bash
kubectl delete deployment inflate
```

---

## จุดที่ต้องระมัดระวังเป็นพิเศษ

### 1. `karpenter` และ `karpenter-crd` ต้องเป็น version เดียวกัน

```text
karpenter-crd 1.14.3
karpenter     1.14.3
```

Version ไม่ตรงกันอาจทำให้ Controller อ่าน field ไม่ได้, validation ปฏิเสธ manifest หรือเกิด `unknown field`

### 2. ระวัง CRD ถูกจัดการโดยหลายระบบ

ห้ามให้ CRD เดียวกันถูกบริหารพร้อมกันโดย Terraform `helm_release`, Helm CLI, Argo CD และ `kubectl apply`

```bash
helm list -A | grep karpenter
kubectl get crd nodepools.karpenter.sh -o jsonpath='{.metadata.annotations.meta\.helm\.sh/release-name}{"\n"}'
kubectl get crd nodepools.karpenter.sh -o jsonpath='{.metadata.annotations.meta\.helm\.sh/release-namespace}{"\n"}'
```

### 3. ห้ามถอน CRD ก่อนถอน Custom Resources

ลำดับถอนที่ปลอดภัย:

1. ปิด workload ที่กระตุ้น scale-out
2. ตรวจสอบ NodeClaims
3. Drain/ลบ Karpenter nodes อย่างควบคุม
4. ลบ NodePools
5. ลบ EC2NodeClasses
6. ถอน `karpenter`
7. ถอน `karpenter-crd` เป็นลำดับสุดท้าย

### 4. Pin AMI ใน Production

ไม่แนะนำ `al2023@latest`; แนะนำ `al2023@vYYYYMMDD` เพื่อลด drift และควบคุมการทดสอบ

### 5. อย่าจำกัด Instance Type แคบเกินไป

ควรเปิดหลาย family, generation และ AZ โดยเฉพาะ Spot ควรมี instance alternatives อย่างน้อยประมาณ 10 ถึง 20 แบบ

### 6. ตรวจสอบ PodDisruptionBudget

```bash
kubectl get pdb -A
```

ระวัง `minAvailable: 100%` หรือ `maxUnavailable: 0` ซึ่งอาจทำให้ Karpenter drain และ consolidate node ไม่ได้

### 7. ตั้ง Resource Requests ให้ Workload

Karpenter ตัดสินใจจาก CPU/memory requests, node selector, affinity, topology spread และ tolerations หาก Pod ไม่ตั้ง requests การ bin-packing และ consolidation อาจไม่ตรงกับการใช้จริง

### 8. ระวัง Subnet IP ไม่พอ

```bash
aws ec2 describe-subnets \
  --filters "Name=tag:karpenter.sh/discovery,Values=${CLUSTER_NAME}" \
  --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,AvailableIPs:AvailableIpAddressCount}' \
  --output table
```

พิจารณา secondary CIDR, VPC CNI prefix delegation, เพิ่ม subnet หรือแยก subnet สำหรับ Karpenter nodes

### 9. EC2 Spot Service-Linked Role

```bash
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com
```

หากไม่มีอาจพบ `ServiceLinkedRoleCreationNotPermitted`

### 10. อย่าใช้ `latest` เป็น Helm version

ควรกำหนด exact version เช่น `--version 1.14.3`

---

## Checklist หลังติดตั้ง

```bash
helm list -n karpenter
kubectl get crd | grep karpenter
kubectl get deployment,pod -n karpenter
kubectl logs -n karpenter deployment/karpenter --all-containers=true --tail=200
kubectl get ec2nodeclass
kubectl describe ec2nodeclass default
kubectl get nodepool
kubectl describe nodepool default
kubectl get nodeclaims
kubectl get nodes -L karpenter.sh/nodepool,karpenter.sh/capacity-type
kubectl get events -A --sort-by=.metadata.creationTimestamp | tail -100
```

สถานะที่คาดหวัง:

```text
Karpenter Deployment: 2/2 Ready
EC2NodeClass: Ready=True
NodePool: Ready=True
NodeClaim: Registered=True, Initialized=True, Ready=True
EC2 Node: Ready
```

---

## สรุปการเลือก Version

สำหรับ **AWS EKS 1.35 Production**:

```text
Kubernetes / EKS : 1.35
karpenter-crd    : 1.14.3
karpenter        : 1.14.3
API              : karpenter.sh/v1
AWS API          : karpenter.k8s.aws/v1
Node OS          : Amazon Linux 2023
AMI              : Pin exact AL2023 release
```

เหตุผลหลักคือ EKS 1.35 ต้องใช้ Karpenter อย่างน้อย 1.9 และ `1.14.3` เป็น LTS ที่รองรับถึงกรกฎาคม 2027 จึงให้สมดุลระหว่าง compatibility, security fixes และความเสถียรสำหรับ Production

---

## แหล่งอ้างอิง

- Karpenter Compatibility: https://karpenter.sh/docs/upgrading/compatibility/
- Karpenter Getting Started: https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/
- Amazon EKS Best Practices - Karpenter: https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html
- Amazon EKS Workshop - Install Karpenter: https://www.eksworkshop.com/docs/fundamentals/compute/karpenter/configure
- Karpenter Troubleshooting: https://karpenter.sh/docs/troubleshooting/
- Karpenter AWS Provider Releases: https://github.com/aws/karpenter-provider-aws/releases
