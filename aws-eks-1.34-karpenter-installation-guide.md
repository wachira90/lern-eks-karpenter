# วิธีติดตั้ง Karpenter บน AWS EKS 1.34

## Version ที่แนะนำ

สำหรับ **Amazon EKS / Kubernetes 1.34**:

- Karpenter ขั้นต่ำตาม Compatibility Matrix คือ **`v1.6.x`**
- สำหรับติดตั้งใหม่ใน Production แนะนำ **Karpenter `v1.14.3 LTS`**
- ติดตั้ง `karpenter` และ `karpenter-crd` เป็น **version เดียวกัน**
- `v1.14.3` รองรับถึงเดือนกรกฎาคม 2027

| Component | Version |
|---|---:|
| Amazon EKS | `1.34` |
| Kubernetes API | `1.34` |
| `karpenter-crd` | `1.14.3` |
| `karpenter` | `1.14.3` |
| Karpenter API | `karpenter.sh/v1` |
| AWS Provider API | `karpenter.k8s.aws/v1` |
| Node OS | Amazon Linux 2023 |
| Helm chart source | `public.ecr.aws/karpenter` |

> แม้ `v1.6.x` จะเป็นขั้นต่ำสำหรับ Kubernetes 1.34 แต่ไม่แนะนำให้เริ่มระบบใหม่ด้วย `1.6.0` สำหรับ Production ควรใช้ LTS release ที่รองรับอยู่และทดสอบแล้ว

---

## 1. กำหนดตัวแปร

คำสั่งต่อไปนี้เหมาะกับ Bash, AWS CloudShell หรือ Linux shell:

```bash
export CLUSTER_NAME="my-eks-cluster"
export AWS_REGION="ap-southeast-1"
export KARPENTER_NAMESPACE="karpenter"
export KARPENTER_VERSION="1.14.3"

export AWS_ACCOUNT_ID="$(
  aws sts get-caller-identity     --query Account     --output text
)"

export KARPENTER_NODE_ROLE="KarpenterNodeRole-${CLUSTER_NAME}"
export KARPENTER_CONTROLLER_ROLE="KarpenterControllerRole-${CLUSTER_NAME}"
export INTERRUPTION_QUEUE="${CLUSTER_NAME}-karpenter"
```

ตรวจสอบค่าที่กำหนด:

```bash
printf 'Account: %s\nCluster: %s\nRegion: %s\nKarpenter: %s\n'   "${AWS_ACCOUNT_ID}"   "${CLUSTER_NAME}"   "${AWS_REGION}"   "${KARPENTER_VERSION}"
```

---

## 2. ตรวจสอบ Cluster ก่อนติดตั้ง

```bash
aws sts get-caller-identity

aws eks describe-cluster   --name "${CLUSTER_NAME}"   --region "${AWS_REGION}"   --query 'cluster.version'   --output text
```

ผลลัพธ์ต้องเป็น `1.34`

```bash
aws eks update-kubeconfig   --name "${CLUSTER_NAME}"   --region "${AWS_REGION}"

kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

ตรวจสอบ add-ons:

```bash
for ADDON in vpc-cni coredns kube-proxy; do
  aws eks describe-addon     --cluster-name "${CLUSTER_NAME}"     --region "${AWS_REGION}"     --addon-name "${ADDON}"     --query 'addon.{Name:addonName,Version:addonVersion,Status:status}'     --output table
done
```

ก่อนติดตั้งต้องแน่ใจว่า:

- `vpc-cni`, `coredns` และ `kube-proxy` รองรับ EKS 1.34
- มี Managed Node Group หรือ Fargate สำหรับรัน Karpenter Controller
- Subnet มี IP address เหลือเพียงพอ
- Cluster endpoint เข้าถึงได้จาก node
- NAT Gateway หรือ VPC endpoints รองรับ AWS APIs ที่จำเป็น

---

## 3. เตรียม System Managed Node Group

Karpenter Controller ไม่ควรพึ่งเฉพาะ node ที่ Karpenter สร้างเอง แนะนำ:

- อย่างน้อย 2 nodes
- กระจายอย่างน้อย 2 Availability Zones
- instance เช่น `m6i.large`, `m7i.large` หรือ `m7g.large`
- Karpenter Controller 2 replicas
- รองรับ CoreDNS, VPC CNI และระบบ monitoring

ตัวอย่าง label:

```bash
kubectl label node <NODE_NAME> workload-type=system
```

ถ้ามี taint:

```bash
kubectl taint node <NODE_NAME> CriticalAddonsOnly=true:NoSchedule
```

ต้องเพิ่ม toleration ให้ Karpenter Controller ใน Helm values ด้วย

---

## 4. Tag Subnet สำหรับ Karpenter

```bash
aws eks describe-cluster   --name "${CLUSTER_NAME}"   --region "${AWS_REGION}"   --query 'cluster.resourcesVpcConfig.subnetIds'   --output text
```

ใส่ tag เฉพาะ private subnet ที่ต้องการใช้:

```bash
aws ec2 create-tags   --region "${AWS_REGION}"   --resources subnet-xxxxxxxx subnet-yyyyyyyy   --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}"
```

ตรวจสอบ IP คงเหลือ:

```bash
aws ec2 describe-subnets   --region "${AWS_REGION}"   --filters "Name=tag:karpenter.sh/discovery,Values=${CLUSTER_NAME}"   --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,AvailableIPs:AvailableIpAddressCount}'   --output table
```

ควรมี private subnet อย่างน้อย 2 AZ และเผื่อ IP สำหรับ EC2 primary ENI, Pod IP, Load Balancer และ future scale-out

---

## 5. Tag Security Group

```bash
export CLUSTER_SECURITY_GROUP="$(
  aws eks describe-cluster     --name "${CLUSTER_NAME}"     --region "${AWS_REGION}"     --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId'     --output text
)"

aws ec2 create-tags   --region "${AWS_REGION}"   --resources "${CLUSTER_SECURITY_GROUP}"   --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}"
```

> ระวังอย่า tag หลาย Security Groups ด้วย discovery value เดียวกันโดยไม่ตั้งใจ

---

## 6. สร้าง IAM Role สำหรับ Karpenter Node

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

aws iam create-role   --role-name "${KARPENTER_NODE_ROLE}"   --assume-role-policy-document file://node-role-trust-policy.json
```

Attach policies:

```bash
aws iam attach-role-policy   --role-name "${KARPENTER_NODE_ROLE}"   --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy   --role-name "${KARPENTER_NODE_ROLE}"   --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly

aws iam attach-role-policy   --role-name "${KARPENTER_NODE_ROLE}"   --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

หาก `aws-node` ยังใช้ Node Role สำหรับ VPC CNI:

```bash
aws iam attach-role-policy   --role-name "${KARPENTER_NODE_ROLE}"   --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
```

หาก `aws-node` ใช้ IRSA หรือ EKS Pod Identity แล้ว ไม่ควรเพิ่ม `AmazonEKS_CNI_Policy` ให้ Node Role โดยไม่จำเป็น

---

## 7. สร้าง EKS Access Entry ให้ Node Role

```bash
aws eks describe-cluster   --name "${CLUSTER_NAME}"   --region "${AWS_REGION}"   --query 'cluster.accessConfig.authenticationMode'   --output text
```

หากใช้ `API` หรือ `API_AND_CONFIG_MAP`:

```bash
aws eks create-access-entry   --cluster-name "${CLUSTER_NAME}"   --region "${AWS_REGION}"   --principal-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${KARPENTER_NODE_ROLE}"   --type EC2_LINUX
```

ตรวจสอบ:

```bash
aws eks describe-access-entry   --cluster-name "${CLUSTER_NAME}"   --region "${AWS_REGION}"   --principal-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${KARPENTER_NODE_ROLE}"
```

ไม่ต้อง associate Cluster Admin policy ให้ Node Role ถ้า Cluster ใช้ `CONFIG_MAP` เพียงอย่างเดียว ต้องเพิ่ม Node Role ใน `aws-auth` หรือเปลี่ยน authentication mode อย่างวางแผนก่อน

---

## 8. สร้าง SQS Interruption Queue

```bash
aws sqs create-queue   --queue-name "${INTERRUPTION_QUEUE}"   --region "${AWS_REGION}"   --attributes '{"MessageRetentionPeriod":"300","SqsManagedSseEnabled":"true"}'

export QUEUE_URL="$(
  aws sqs get-queue-url     --queue-name "${INTERRUPTION_QUEUE}"     --region "${AWS_REGION}"     --query QueueUrl     --output text
)"

export QUEUE_ARN="$(
  aws sqs get-queue-attributes     --queue-url "${QUEUE_URL}"     --region "${AWS_REGION}"     --attribute-names QueueArn     --query 'Attributes.QueueArn'     --output text
)"
```

ต้องสร้าง EventBridge Rules สำหรับ:

1. EC2 Spot Instance Interruption Warning
2. EC2 Instance Rebalance Recommendation
3. EC2 Instance State-change Notification
4. AWS Health Scheduled Change

พร้อม EventBridge target, SQS Queue Policy และสิทธิ์ให้ Controller อ่าน/ลบ message แนะนำใช้ Terraform หรือ CloudFormation เพื่อป้องกันองค์ประกอบตกหล่น

---

## 9. สร้าง Controller IAM Policy

Controller ต้องมีสิทธิ์หลัก เช่น:

```text
ec2:CreateFleet
ec2:RunInstances
ec2:CreateLaunchTemplate
ec2:DeleteLaunchTemplate
ec2:TerminateInstances
ec2:CreateTags
ec2:Describe*
ssm:GetParameter
pricing:GetProducts
eks:DescribeCluster
iam:PassRole
sqs:GetQueueUrl
sqs:GetQueueAttributes
sqs:ReceiveMessage
sqs:DeleteMessage
```

จำกัด `iam:PassRole` ให้ชี้ไปที่ Node Role เท่านั้น:

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/KarpenterNodeRole-<CLUSTER_NAME>"
}
```

หลีกเลี่ยง `Resource: "*"` และควรสร้าง Controller Policy จาก policy template ของ Karpenter version ที่ติดตั้ง

---

## 10. ผูก Controller Role ด้วย IRSA

```bash
eksctl utils associate-iam-oidc-provider   --cluster "${CLUSTER_NAME}"   --region "${AWS_REGION}"   --approve

eksctl create iamserviceaccount   --cluster "${CLUSTER_NAME}"   --region "${AWS_REGION}"   --namespace "${KARPENTER_NAMESPACE}"   --name karpenter   --role-name "${KARPENTER_CONTROLLER_ROLE}"   --attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}"   --role-only   --approve

export KARPENTER_CONTROLLER_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${KARPENTER_CONTROLLER_ROLE}"
```

---

## 11. ตรวจสอบว่าไม่มี Karpenter เดิม

```bash
helm list -A | grep -i karpenter || true
kubectl get crd | grep -E 'karpenter.sh|karpenter.k8s.aws' || true
kubectl get nodepool,ec2nodeclass,nodeclaim --ignore-not-found
```

ห้ามติดตั้งทับถ้า CRD เดิมถูกบริหารโดย Terraform, Argo CD หรือ Helm release อื่น

---

## 12. ติดตั้ง `karpenter-crd`

```bash
aws ecr-public get-login-password   --region us-east-1 | helm registry login   --username AWS   --password-stdin public.ecr.aws

kubectl create namespace "${KARPENTER_NAMESPACE}"   --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install karpenter-crd   oci://public.ecr.aws/karpenter/karpenter-crd   --version "${KARPENTER_VERSION}"   --namespace "${KARPENTER_NAMESPACE}"   --wait   --timeout 10m
```

ตรวจสอบ:

```bash
helm status karpenter-crd --namespace "${KARPENTER_NAMESPACE}"
kubectl get crd | grep -E 'karpenter.sh|karpenter.k8s.aws'
```

ต้องพบอย่างน้อย:

```text
nodepools.karpenter.sh
nodeclaims.karpenter.sh
ec2nodeclasses.karpenter.k8s.aws
```

---

## 13. สร้าง Helm values สำหรับ Controller

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

แทนค่า:

```bash
sed -i   "s|REPLACE_CONTROLLER_ROLE_ARN|${KARPENTER_CONTROLLER_ROLE_ARN}|g;
   s|REPLACE_CLUSTER_NAME|${CLUSTER_NAME}|g;
   s|REPLACE_INTERRUPTION_QUEUE|${INTERRUPTION_QUEUE}|g"   karpenter-values.yaml
```

หากบังคับ Controller ให้อยู่บน system nodes เพิ่ม:

```yaml
nodeSelector:
  workload-type: system
```

หาก system node มี taint เพิ่ม:

```yaml
tolerations:
  - key: CriticalAddonsOnly
    operator: Exists
```

---

## 14. Render และติดตั้ง Controller

```bash
helm template karpenter   oci://public.ecr.aws/karpenter/karpenter   --version "${KARPENTER_VERSION}"   --namespace "${KARPENTER_NAMESPACE}"   --values karpenter-values.yaml   > karpenter-rendered.yaml

helm upgrade --install karpenter   oci://public.ecr.aws/karpenter/karpenter   --version "${KARPENTER_VERSION}"   --namespace "${KARPENTER_NAMESPACE}"   --values karpenter-values.yaml   --wait   --timeout 15m   --atomic
```

ตรวจสอบ:

```bash
kubectl rollout status deployment/karpenter   --namespace "${KARPENTER_NAMESPACE}"   --timeout 5m

kubectl get deployment,pod,serviceaccount   --namespace "${KARPENTER_NAMESPACE}"   -o wide

kubectl logs   --namespace "${KARPENTER_NAMESPACE}"   deployment/karpenter   --all-containers=true   --tail=200
```

---

## 15. เลือก AL2023 AMI สำหรับ EKS 1.34

```bash
aws ssm get-parameter   --region "${AWS_REGION}"   --name "/aws/service/eks/optimized-ami/1.34/amazon-linux-2023/x86_64/standard/recommended/image_id"   --query 'Parameter.Value'   --output text

aws ssm get-parameter   --region "${AWS_REGION}"   --name "/aws/service/eks/optimized-ami/1.34/amazon-linux-2023/x86_64/standard/recommended/release_version"   --query 'Parameter.Value'   --output text
```

Production ควร pin AMI release ที่ทดสอบแล้วแทน `@latest`

---

## 16. สร้าง `EC2NodeClass`

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
    - alias: al2023@REPLACE_TESTED_RELEASE

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

  tags:
    Name: REPLACE_CLUSTER_NAME-karpenter
    karpenter.sh/discovery: REPLACE_CLUSTER_NAME
    Environment: production
```

Apply และตรวจสอบ:

```bash
kubectl apply -f ec2nodeclass.yaml
kubectl get ec2nodeclass
kubectl describe ec2nodeclass default
```

ต้องรอ `Ready=True`

---

## 17. สร้าง NodePool

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general
spec:
  template:
    metadata:
      labels:
        workload-type: general
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
kubectl describe nodepool general
```

ต้องรอ `Ready=True`

---

## 18. ทดสอบ Scale-out

สร้าง `inflate.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      nodeSelector:
        workload-type: general
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.10
          resources:
            requests:
              cpu: "1"
              memory: 1Gi
```

```bash
kubectl apply -f inflate.yaml
kubectl scale deployment inflate --replicas=10
kubectl get pods -w
kubectl get nodeclaims -w
kubectl get nodes -L karpenter.sh/nodepool,karpenter.sh/capacity-type
kubectl logs -n "${KARPENTER_NAMESPACE}" deployment/karpenter --all-containers=true --follow
```

หลังทดสอบ:

```bash
kubectl scale deployment inflate --replicas=0
kubectl get nodeclaims -w
kubectl delete deployment inflate
```

---

# จุดที่ต้องระวังสำหรับ EKS 1.34

## 1. อย่าเลือก Karpenter `1.6.0` เพียงเพราะเป็นขั้นต่ำ

สำหรับติดตั้งใหม่ใช้:

```text
karpenter-crd 1.14.3
karpenter     1.14.3
```

หากจำเป็นต้องอยู่สาย `1.6.x` ให้ใช้ patch ล่าสุดในสาย `1.6` และทดสอบ compatibility log อย่างละเอียด

## 2. Controller และ CRD ต้องตรง version

ไม่ควรใช้ `karpenter-crd 1.6.8` คู่กับ `karpenter 1.14.3`

## 3. อย่าให้ CRD มีหลาย owner

เลือกเจ้าของเพียงระบบเดียว เช่น Terraform, Helm CLI, Argo CD หรือ Flux

## 4. Pin AL2023 AMI สำหรับ EKS 1.34

ไม่แนะนำ `al2023@latest` ใน Production แนะนำ `al2023@vYYYYMMDD` ที่ผ่าน staging แล้ว

## 5. ห้ามลบ CRD ก่อน NodePool และ NodeClaim

ลำดับถอนที่ปลอดภัย:

1. หยุด workload ที่ทำให้ scale-out
2. ตรวจสอบ NodeClaim
3. Drain และลบ Karpenter nodes อย่างควบคุม
4. ลบ NodePool
5. ลบ EC2NodeClass
6. ถอน `karpenter`
7. ถอน `karpenter-crd` เป็นลำดับสุดท้าย

## 6. ตรวจสอบ Access Entry หรือ `aws-auth`

ถ้า EC2 ถูกสร้างแต่ไม่ join cluster ให้ตรวจสอบ authentication mode, Access Entry, Security Group, NAT/VPC endpoint, AMI, DNS และ VPC CNI

## 7. เปิด instance type ให้กว้างพอ

สำหรับ Spot ควรมีทางเลือกอย่างน้อย 10 ถึง 20 instance types กระจายหลาย family และหลาย AZ

## 8. ตรวจสอบ PodDisruptionBudget

```bash
kubectl get pdb -A
```

`minAvailable: 100%` หรือ `maxUnavailable: 0` อาจบล็อก consolidation

## 9. กำหนด requests ให้ Pod

Karpenter ใช้ CPU/memory requests, node selectors, affinity, tolerations และ topology constraints ในการเลือก instance

## 10. ตรวจสอบ IP ใน Subnet

พิจารณา VPC CNI Prefix Delegation, secondary VPC CIDR, เพิ่ม private subnet หรือแยก subnet สำหรับ Karpenter เมื่อ IP เหลือน้อย

---

# Checklist หลังติดตั้ง

```bash
helm list -n karpenter
kubectl get crd | grep -E 'karpenter.sh|karpenter.k8s.aws'
kubectl get deployment,pod -n karpenter -o wide
kubectl logs -n karpenter deployment/karpenter --all-containers=true --tail=200
kubectl get ec2nodeclass
kubectl describe ec2nodeclass default
kubectl get nodepool
kubectl describe nodepool general
kubectl get nodeclaims -o wide
kubectl get nodes -L karpenter.sh/nodepool,karpenter.sh/capacity-type
kubectl get events -A --sort-by=.metadata.creationTimestamp | tail -100
```

สถานะที่ต้องได้:

```text
Karpenter Deployment: 2/2 Ready
EC2NodeClass/default: Ready=True
NodePool/general: Ready=True
NodeClaim: Registered=True
NodeClaim: Initialized=True
NodeClaim: Ready=True
EC2 Node: Ready
```

---

# สรุป

```text
EKS               : 1.34
karpenter-crd     : 1.14.3
karpenter         : 1.14.3
EC2NodeClass API  : karpenter.k8s.aws/v1
NodePool API      : karpenter.sh/v1
Node OS           : Amazon Linux 2023
AMI               : Pin release ที่รองรับ EKS 1.34
Controller        : 2 replicas บน Managed Node Group
```

แม้ Karpenter ขั้นต่ำสำหรับ Kubernetes 1.34 คือ `v1.6` แต่ `v1.14.3 LTS` เหมาะกว่าสำหรับการติดตั้งใหม่ เพราะเป็น LTS และได้รับ bug/security fixes ที่ใหม่กว่า

---

## แหล่งอ้างอิง

- [Karpenter Compatibility](https://karpenter.sh/docs/upgrading/compatibility/)
- [Karpenter AWS Provider Releases](https://github.com/aws/karpenter-provider-aws/releases)
- [Amazon EKS Best Practices: Karpenter](https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html)
- [Karpenter Managing AMIs](https://karpenter.sh/docs/tasks/managing-amis/)
