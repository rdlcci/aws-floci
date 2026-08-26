# SKILL.md — AWS Systems Intuition Lab (Floci Edition)

## Purpose

This is not a "deploy and forget" AWS tutorial repo. The goal is to build **mechanical
intuition** for how AWS components interact — what breaks, what slows down, what gets
more resilient, and what gets more expensive, when you add/remove/misconfigure a piece.

Every module in this repo follows the same loop:
1. **Provision** the component manually (CLI first, IaC later).
2. **Break it on purpose** (remove it, misconfigure it, starve it).
3. **Measure** the effect (latency, errors, availability, cost-in-theory).
4. **Write down** what you observed in that module's `NOTES.md`.

If a module doesn't end with you being able to explain "why would I add this / what do
I lose without it" in one paragraph, it's not done.

---

## How to use this file

Whenever you (or an assistant helping you) start a new module, re-read the relevant
section below first. Each section defines:
- **What it is / where it sits in the stack**
- **Dependencies** (what must exist before this can)
- **Break-it experiments** (the actual point of the module)
- **Floci gotchas** (things that behave differently from real AWS or aren't supported)
- **Definition of done**

---

## Repo Structure

```
aws-floci-lab/
├── SKILL.md                     # this file
├── docker-compose.yml           # spins up Floci
├── 00-foundations/
│   ├── iam/
│   ├── vpc-networking/
│   └── README.md
├── 01-compute/
│   ├── ec2-single-instance/
│   ├── ebs-volumes-and-snapshots/
│   ├── ami-custom-images/
│   ├── auto-scaling-groups/
│   └── README.md
├── 02-containers/
│   ├── ecr-registry/
│   ├── ecs-fargate/
│   ├── ecs-ec2-launch-type/
│   ├── eks-basics/
│   └── README.md
├── 03-storage/
│   ├── s3-buckets/
│   ├── s3-lifecycle-versioning/
│   ├── efs-shared-filesystem/
│   └── README.md
├── 04-databases/
│   ├── rds-postgres/
│   ├── dynamodb/
│   ├── elasticache-redis/
│   └── README.md
├── 05-networking-edge/
│   ├── nat-gateway-vs-nat-instance/
│   ├── internet-gateway/
│   ├── vpc-endpoints/
│   ├── vpc-peering-transit-gateway/
│   ├── alb-vs-nlb/
│   ├── api-gateway/
│   ├── cloudfront-cdn/
│   ├── waf-and-shield/
│   ├── route53-dns/
│   ├── bastion-vs-ssm-session-manager/
│   └── README.md
├── 06-messaging-events/
│   ├── sqs-queues/
│   ├── sns-topics/
│   ├── eventbridge/
│   ├── kinesis-data-streams/
│   └── README.md
├── 07-serverless/
│   ├── lambda-basics/
│   ├── lambda-vpc-attached/
│   ├── step-functions/
│   └── README.md
├── 08-observability/
│   ├── cloudwatch-metrics-logs/
│   ├── cloudwatch-alarms/
│   ├── cloudtrail-audit-logging/
│   ├── xray-tracing/
│   └── README.md
├── 09-security-secrets/
│   ├── security-groups-vs-nacls/
│   ├── kms-encryption/
│   ├── secrets-manager/
│   ├── ssm-parameter-store/
│   ├── sts-assume-role/
│   └── README.md
├── 10-architectures/            # composed multi-component systems
│   ├── 3-tier-webapp/
│   ├── with-and-without-cache/
│   ├── with-and-without-lb/
│   ├── multi-az-failover/
│   └── README.md
├── 11-cicd-iac/
│   ├── codepipeline-codebuild-codedeploy/
│   ├── cloudformation-vs-terraform/
│   └── README.md
├── terraform/                   # IaC versions, once manual is understood
│   └── modules/
├── load-testing/
│   ├── locustfile.py
│   └── results/
└── docs/
    └── decision-log.md          # "I added X because Y" running journal
```

Each leaf folder contains:
- `README.md` — what this component is, dependencies, break-it experiments
- `provision.sh` — the raw AWS CLI commands (against Floci)
- `NOTES.md` — your observations after running the experiments
- `cleanup.sh` — teardown

---

## Module Reference

### 00 — Foundations

**IAM**
- What: identities, roles, policies — everything else depends on this conceptually,
  even though Floci won't enforce most IAM restrictions.
- Break-it: attach an overly narrow policy to a role and watch an API call fail with
  `AccessDenied`; broaden it and re-test.
- Done when: you can explain role vs. user vs. policy vs. trust relationship without
  looking it up.

**VPC Networking**
- What: VPC, subnets (public/private), route tables, CIDR blocks. This is the
  substrate everything else (EC2, RDS, Lambda-in-VPC, ALB) sits inside.
- Dependencies: none — this is the floor.
- Break-it:
  - Launch an EC2 instance in a subnet with **no route to an Internet Gateway** →
    it can't reach the internet even with a public IP.
  - Put two subnets in different AZs, kill one AZ's route table → see what's isolated.
- Done when: you can draw (on paper) a VPC with 2 public + 2 private subnets across
  2 AZs, with route tables, from memory.

---

### 01 — Compute

**EC2 (single instance)**
- What: the "VPS" — a virtual machine.
- Dependencies: VPC, subnet, security group, key pair, AMI.
- Break-it: remove the security group inbound rule for port 80 → app unreachable
  even though it's running. This teaches SG vs. "is my app actually up" debugging.

**EBS (volumes & snapshots)**
- What: persistent block storage attached to EC2. Ephemeral vs. persistent data.
- Break-it: stop/start an instance (data on EBS persists) vs. what would happen with
  instance store (data gone). Take a snapshot, delete the volume, restore from
  snapshot.
- Done when: you can explain why a database should never live on instance-store-only.

**AMI (custom images)**
- What: baking a "golden image" so instances boot pre-configured instead of
  bootstrapping every time.
- Break-it: time a boot-and-configure-via-script instance vs. a pre-baked AMI. This
  is the intuition for why AMIs matter at scale / in auto-scaling.

**Auto Scaling Groups**
- What: automatic add/remove of EC2 instances based on demand or health.
- Dependencies: launch template/config, (usually) a load balancer target group.
- Break-it: kill an instance manually → watch ASG replace it. Load test → watch it
  scale out. Set a low max size → watch it hit a ceiling under load.

---

### 02 — Containers

**ECR (registry)**
- What: private Docker registry. Where your images live before ECS/EKS/Lambda use them.
- Break-it: push an image, then try to pull it with a role that lacks
  `ecr:GetDownloadUrlForLayer` permission.

**ECS — Fargate vs EC2 launch type**
- What: Fargate = serverless containers (AWS manages the host). EC2 launch type =
  you manage the underlying instances the containers run on.
- Break-it: compare cold-start time and control tradeoffs between the two. Kill a
  task manually in both and observe ECS service scheduler behavior.
- Floci gotcha: check current Floci docs for Fargate networking parity — container
  orchestration fidelity varies by emulator version.

**EKS (basics)**
- What: managed Kubernetes control plane. Compare directly against ECS — same
  problem (run containers, don't manage the orchestrator), different API/mental
  model.
- Dependencies: VPC (EKS wants its own subnet tagging conventions), IAM roles for
  service accounts (IRSA).
- Break-it: deploy the same app as an ECS service and as a Kubernetes Deployment on
  EKS. Kill a pod vs. kill an ECS task — compare how each control plane notices and
  replaces it.
- Floci gotcha: full EKS control-plane emulation is heavier than ECS in most local
  emulators — check current service coverage before investing a lot of time here;
  a lot of the *conceptual* learning (Deployments, Services, ReplicaSets) can happen
  against `kind`/`minikube` directly if EKS-specific parity is thin.
- Done when: you can explain, out loud, what ECS gives you that EKS doesn't and
  vice versa — not just "EKS is Kubernetes."

---

### 03 — Storage

**S3 (buckets)**
- What: object storage. No "servers" — this is the first fully-managed service you'll
  touch.
- Break-it: make a bucket private, try to `curl` an object directly → 403. Add a
  bucket policy allowing public read → retry.

**S3 lifecycle & versioning**
- What: automatic tiering (Standard → IA → Glacier) and object versioning for
  accidental-delete protection.
- Break-it: enable versioning, delete an object, recover the previous version.
  Without versioning, repeat — it's gone.

**EFS (shared filesystem)**
- What: managed NFS — a filesystem multiple EC2/ECS/Lambda instances can mount
  *simultaneously*, unlike EBS (which attaches to one instance at a time, in one AZ).
- Dependencies: mount targets in each subnet/AZ you want access from; security group
  allowing NFS (2049) from your compute's SG.
- Break-it: mount the same EFS volume from two EC2 instances, write a file from one,
  read it from the other immediately. Then try the same thing with EBS (you can't —
  it's single-attach) to feel the actual difference, not just recite it.
- Done when: you can say in one sentence when you'd reach for EFS over EBS (shared
  access across multiple compute nodes) vs. S3 (EFS behaves like a real filesystem
  with POSIX semantics; S3 is object storage accessed via API, not mounted as a
  drive).

---

### 04 — Databases

**RDS (Postgres/MySQL)**
- What: managed relational database. Compare to running Postgres yourself on EC2.
- Dependencies: VPC subnet group (usually private subnets), security group allowing
  inbound from your app's SG only.
- Break-it: connect from an EC2 instance in the same VPC (works) vs. from your local
  machine directly (blocked unless you open it up — and understand why you shouldn't).

**DynamoDB**
- What: managed NoSQL, contrast against RDS for access-pattern-driven design.
- Break-it: model the same data both ways (relational join vs. single-table NoSQL
  design) and feel the difference in query flexibility vs. scalability.

**ElastiCache (Redis)**
- What: in-memory cache in front of a database.
- Dependencies: sits in the same VPC as your app; app must be coded to check cache
  before hitting DB.
- **This is your headline experiment.** Build the same endpoint two ways:
  - No cache: every request hits RDS directly.
  - With cache: cache-aside pattern (check Redis → miss → query RDS → populate Redis).
  - Load test both. Measure p50/p95 latency and RDS CPU/connections under load.
  - Then kill Redis mid-test → watch what "cache down" does to your app (does it
    fail gracefully or fall over?).

---

### 05 — Networking & Edge

**NAT Gateway vs NAT Instance**
- What: lets private-subnet resources reach the internet **outbound only** (e.g., to
  pull OS updates) without being reachable from the internet.
- Break-it: private-subnet EC2 with no NAT → `yum update` hangs/fails. Add NAT
  Gateway → works. Compare NAT Gateway (managed, HA, $) vs. NAT Instance (an EC2 you
  manage, single point of failure unless you build HA yourself).

**Internet Gateway**
- What: the only way traffic enters/leaves a VPC to/from the public internet.
- Break-it: public subnet + public IP but no IGW attached/no route to it → still
  unreachable. This is the #1 real-world "why can't I reach my instance" bug.

**VPC Endpoints (Gateway & Interface)**
- What: private connectivity from inside your VPC directly to AWS services (S3,
  DynamoDB via Gateway endpoints; most other services via Interface/PrivateLink
  endpoints) — **without** routing through a NAT Gateway or the public internet at
  all.
- Dependencies: sits in your private subnets' route tables (Gateway) or as an ENI
  with a security group (Interface).
- Break-it: from a private subnet with only a NAT Gateway, call S3 — watch it work
  but cost NAT data-processing charges. Add an S3 Gateway Endpoint, remove the NAT
  route for S3 traffic specifically, call S3 again — same result, but now it never
  left the AWS network and costs nothing extra. This is the single most common
  "wait, I didn't need NAT for that" realization in real AWS bills.
- Done when: you can explain why a private-subnet Lambda talking only to S3/DynamoDB
  never needs a NAT Gateway if it has the right endpoints.

**VPC Peering & Transit Gateway**
- What: connecting two or more VPCs so resources in each can talk to each other
  privately. Peering = point-to-point, non-transitive. Transit Gateway = hub-and-spoke,
  scales to many VPCs.
- Break-it: peer two VPCs, confirm private IP connectivity works, then try to reach
  a *third* VPC peered to one of them but not the other — it fails (peering isn't
  transitive). That failure is the entire reason Transit Gateway exists at scale.

**ALB vs NLB**
- What: ALB = layer 7 (HTTP-aware routing, path/host-based rules). NLB = layer 4
  (raw TCP/UDP, ultra-low latency, static IP).
- Break-it: put 2 EC2s behind ALB, watch round-robin/health-check-based routing.
  Kill one instance's health check → traffic stops going to it. Compare NLB when you
  need a static IP or non-HTTP protocol.

**API Gateway**
- What: managed front door for APIs — especially Lambda. Handles throttling, auth,
  request/response transformation.
- Break-it: set a low throttle limit, hammer it, watch 429s.

**CloudFront (CDN)**
- What: edge caching in front of S3/ALB/API Gateway.
- Break-it: time a request to origin vs. cached-at-edge on second request. Invalidate
  cache, see the next request go to origin again.

**WAF & Shield**
- What: WAF = layer-7 firewall rules (block SQLi patterns, rate-limit an IP,
  geo-block) attached to CloudFront/ALB/API Gateway. Shield = DDoS protection
  (Standard is automatic/free; Advanced is paid and detailed).
- Break-it: attach a WAF rule rate-limiting a path to 5 req/min, hammer it past that,
  watch it start returning 403s before the request even reaches your app.

**Route53 (DNS)**
- What: DNS + health-check-based failover/routing policies.
- Break-it: set up a failover routing policy between two endpoints, kill the
  "primary," watch DNS resolve to secondary (and how slow/fast that actually is —
  DNS TTL matters here).

**Bastion Host vs. SSM Session Manager**
- What: two ways to get a shell on a private-subnet instance. Bastion = a public
  EC2 you SSH into first, then hop from. Session Manager = SSM agent on the instance,
  no open inbound port, no bastion, no key management, full audit trail via CloudTrail.
- Break-it: set up a bastion (open SSH port, manage a key pair) for a private
  instance, then do the same access via Session Manager with zero inbound rules.
  Compare the attack surface and the operational overhead directly.
- Done when: you can explain why most modern AWS setups have moved away from bastion
  hosts.

---

### 06 — Messaging & Events

**SQS**
- What: decoupling — a queue between a producer and consumer so they don't need to
  be up at the same time.
- Break-it: stop your consumer, keep producing → watch queue depth grow instead of
  requests failing. This is the core intuition for "why queues instead of direct
  calls."

**SNS**
- What: pub/sub fan-out — one message, multiple subscribers (including SQS queues).
- Break-it: add a second subscriber to a topic without touching the producer at all.

**EventBridge**
- What: event bus with routing rules based on event content, not just topic name.
- Break-it: write two rules with overlapping patterns, see both fire; narrow the
  pattern, see only one fire.

**Kinesis (Data Streams)**
- What: high-throughput, ordered streaming — the answer when SQS's ordering/
  throughput/fan-out-with-replay limits stop being enough (e.g., clickstream data,
  metrics pipelines, multiple independent consumers replaying the same stream).
- Dependencies: producers write records with a partition key; consumers (including
  Lambda) read via shard iterators.
- Break-it: compare against SQS directly — with Kinesis, add a *second* consumer
  application that reads the same events independently (impossible with a standard
  SQS queue, where one message is consumed once). Also: shard count caps throughput —
  push past it deliberately and watch `ProvisionedThroughputExceeded` errors.
- Done when: you can state the one-line rule of thumb — "SQS for decoupling a
  producer from a consumer; Kinesis for multiple consumers replaying an ordered,
  high-throughput stream."

---

### 07 — Serverless

**Lambda (basics)**
- What: run code without provisioning servers, billed per invocation/duration.
- Break-it: measure cold start vs. warm start latency. Set memory low → watch
  timeouts on heavier workloads; raise memory (which also raises CPU) → compare.

**Lambda in a VPC**
- What: needed when Lambda must reach RDS/ElastiCache inside a VPC — but this adds
  ENI attachment overhead.
- Break-it: compare cold start of a non-VPC Lambda vs. a VPC-attached one.

**Step Functions**
- What: orchestrates multiple Lambdas/services as an explicit state machine (retry,
  branch, parallel, wait) instead of chaining them with brittle code.
- Break-it: build the same 3-step workflow two ways — (a) one Lambda calling the
  next two directly in code, (b) a Step Functions state machine calling three
  Lambdas. Force a failure in step 2 in both — observe how much harder it is to see
  *where* and *why* it failed in version (a) vs. the visual execution history in (b).

---

### 08 — Observability

**CloudWatch metrics & logs**
- What: the nervous system — without this, every other break-it experiment above is
  just "vibes." Wire this up early and revisit it in every later module.
- Break-it: after building the cache/no-cache experiment in module 04, don't just
  observe latency manually — pull the actual CloudWatch metrics and graph them.

**CloudWatch Alarms**
- What: automated reaction to metrics (e.g., trigger ASG scale-out, notify via SNS).
- Break-it: set an alarm threshold, cross it deliberately, confirm the action fires.

**CloudTrail (audit logging)**
- What: records every API call made against your account — who did what, when,
  from where. This is *distinct* from CloudWatch Logs (which is app/service output).
  People conflate these two constantly; don't be one of them.
- Break-it: delete an S3 bucket via the CLI, then find that exact action in
  CloudTrail. Ask yourself: could you answer "who terminated this EC2 instance and
  when" using only CloudWatch? (No — that's CloudTrail's job.)

**X-Ray (tracing)**
- What: distributed tracing — follows a single request across multiple services
  (API Gateway → Lambda → DynamoDB, say) and shows you exactly which hop was slow.
- Dependencies: your app/Lambda needs the X-Ray SDK instrumented, or auto-instrumentation
  enabled where supported.
- Break-it: in your 3-tier app from module 10, artificially add latency to one
  downstream call (e.g., `sleep(2)` in the DB layer) and confirm X-Ray's trace map
  correctly identifies that hop as the bottleneck, not just "the request was slow."

---

### 09 — Security & Secrets

**Security Groups vs NACLs**
- What: SGs are stateful, instance-level; NACLs are stateless, subnet-level.
- Break-it: allow inbound 443 in SG but deny it in the subnet's NACL → still blocked.
  This is the classic "I opened the port but it still doesn't work" bug, on purpose.

**KMS**
- What: encryption key management underlying EBS/S3/RDS encryption-at-rest.
- Break-it: encrypt an EBS volume, try to share/copy it across accounts without key
  permissions → fails.

**Secrets Manager**
- What: don't hardcode DB passwords. Compare app config with env-var secrets vs.
  Secrets Manager retrieval at runtime.

**SSM Parameter Store**
- What: the free/simpler sibling to Secrets Manager — for config that isn't
  sensitive (feature flags, non-secret settings) or secrets that don't need
  automatic rotation.
- Break-it: store the same value in both Parameter Store and Secrets Manager, note
  the cost/rotation/versioning differences, and write down a one-line rule for which
  one you'd pick for a given value (rotation needed + highly sensitive → Secrets
  Manager; everything else → Parameter Store).

**STS / AssumeRole**
- What: temporary, short-lived credentials — the mechanism behind cross-account
  access and "an EC2 instance's role" actually working under the hood.
- Break-it: have one role assume another role with a narrower policy, confirm the
  resulting temporary credentials can only do what the *narrower* role allows, even
  though the original identity had broader permissions. This is the core intuition
  for least-privilege delegation.

---

### 10 — Composed Architectures (the payoff)

This is where modules 00–09 get assembled into real systems, and where your original
question — "what if load balancer is there, what if cache is not" — gets answered
directly and empirically.

**3-tier webapp**: ALB → EC2/ECS (app tier) → RDS (data tier), private subnets for
app+db, public subnet only for the ALB.

**With and without cache**: same app, run load test with ElastiCache in the path and
without. Produce a before/after latency graph. This is your "cache" answer.

**With and without load balancer**: single EC2 vs. ALB + 2 EC2s behind it. Kill one
instance in each setup and compare downtime. This is your "load balancer" answer.

**Multi-AZ failover**: RDS Multi-AZ, ASG spanning 2 AZs behind ALB. Simulate an AZ
outage (stop all resources in one AZ) and time recovery.

Each of these gets a `RESULTS.md` with actual numbers (latency, error rate, recovery
time) — not just "yeah it seemed faster."

---

### 11 — CI/CD & IaC

You've been provisioning everything by hand or with Terraform run manually so far.
This module closes the loop on "how does code actually get from a git push to a
running service."

**CodePipeline / CodeBuild / CodeDeploy**
- What: source (git push) → build (compile/test/package, CodeBuild) → deploy
  (roll out to EC2/ECS/Lambda, CodeDeploy), orchestrated end-to-end by CodePipeline.
- Break-it: push a change, watch it flow through each stage. Then break the build
  deliberately (failing test) and confirm the pipeline halts before deploy — this is
  the entire point of having stages instead of a single deploy script.

**CloudFormation vs. Terraform**
- What: you've been using Terraform since module 00-ish; CloudFormation is AWS's
  native IaC equivalent. Worth a short module purely to compare state management
  (Terraform's state file vs. CloudFormation's managed stack state) and drift
  detection.
- Break-it: manually change a resource that IaC created (e.g., resize an EBS volume
  in the console/CLI outside of Terraform), then run `terraform plan` and a
  CloudFormation drift-detection check — compare how each surfaces the drift.

---

## Deliberately Out of Scope (for now)

Not every AWS service belongs in this lab — some don't emulate meaningfully locally,
some are legacy, and some are narrow enough to learn on-demand later rather than
front-load here:

- **Elastic Beanstalk** — legacy PaaS; ECS/Fargate/EKS cover the same ground and are
  what you'll actually encounter in most modern stacks.
- **Direct Connect** — hybrid on-prem-to-AWS networking; not meaningfully testable
  without real on-prem infrastructure, and low value for this exercise specifically.
- **Cognito** — auth-as-a-service; worth knowing it exists, but narrow enough that
  it's better learned when a specific project needs user sign-in, rather than as
  part of the core infra loop.
- **Data/analytics services** (Athena, Glue, EMR, Redshift) — a different track
  entirely (data engineering vs. systems architecture). Revisit once the core loop
  here is solid, as its own module group if that direction interests you.

If you find yourself reaching for one of these mid-project, that's a sign the
project has grown past this repo's original scope — good, add a module for it then.

---

## Tooling Conventions

- **CLI first, Terraform second.** Don't write `.tf` for a component until you've
  provisioned it manually via `awslocal`/`aws --endpoint-url` at least once.
- **Load testing**: use `locust` (Python, easy to script realistic user behavior) for
  anything in module 10. Save raw results under `load-testing/results/<date>-<test>/`.
- **Every teardown matters**: `cleanup.sh` in each module should fully tear down what
  `provision.sh` created — treat Floci like real AWS in terms of hygiene, even though
  it's free, so the habit transfers.
- **Floci gotchas**: keep a running list in `docs/decision-log.md` of anything that
  behaved differently than real AWS would — Floci is an actively evolving emulator,
  so re-check current service coverage before assuming parity, especially for newer
  or more obscure services.

---

## Definition of Done (per module)

A module is complete only when its `NOTES.md` answers, in your own words:
1. What does this component actually do, mechanically?
2. What depends on it, and what does it depend on?
3. What did you observe when you removed/broke it?
4. When would you actually reach for this in a real system, and when would you not?

If you can't answer #4, you've memorized the CLI command, not the concept — go back
and break it differently.