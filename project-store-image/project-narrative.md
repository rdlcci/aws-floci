# Project Narrative — "Pixly" (a photo-sharing app that outgrows itself)

## Why this doc exists

SKILL.md gives you the component-by-component curriculum. This doc gives you the
**story** that makes each component's *reason to exist* obvious instead of arbitrary.
Build Pixly stage by stage, in order. Don't skip ahead — the whole point is that each
new component is the answer to a pain the previous stage actually caused.

At each stage: build it, load-test/break it as described, feel the pain, *then* read
the "why this fixes it" note, *then* move to the next stage. If you add a component
before you've felt the pain it solves, you'll memorize the CLI command and forget the
reason within a week.

Rule for this exercise: whenever a stage says "pain," actually reproduce it (load
test, kill something, check a real metric) before reading the fix. Don't take the
narrative's word for it.

---

## Stage 0 — MVP

**Build:** One EC2 instance running your app (any stack — Flask/Express/whatever).
Images saved to local disk. Metadata (captions, users) in SQLite on the same disk.

**Components introduced:** EC2, EBS, VPC basics, Security Groups.
*(Module: 00-foundations, 01-compute)*

**It works fine for you and three friends testing it.**

---

## Stage 1 — "I restarted the instance and lost everyone's photos"

**Pain:** You stop/start the instance for a patch. Wait — actually EBS persists
across stop/start, so this specific pain requires you to *terminate and relaunch*
(new instance = new root volume) or simulate a disk-full scenario. Either way: local
disk storage is a single point of failure and doesn't scale past one box's disk size.

**Fix:** Move image storage to **S3**. Now images survive independent of any
instance's lifecycle, and storage is effectively unlimited.

**Component introduced:** S3.
*(Module: 03-storage)*

---

## Stage 2 — "Two people posted at the same time and the app locked up"

**Pain:** SQLite on local disk chokes under concurrent writes (file-level locking).
You also want real queries — "show me the 20 most recent posts from people I
follow" — which SQLite-on-a-single-file starts to strain under.

**Fix:** Move metadata to **RDS Postgres** in a private subnet. App (still on EC2,
public subnet) connects to RDS (private subnet) — this is your first real
public/private subnet split with a purpose behind it, not just a diagram convention.

**Components introduced:** RDS, private subnets (reinforces VPC module), Security
Group rules scoped to "app SG → RDS SG only."

---

## Stage 3 — "A post went viral and the site fell over"

**Pain:** Single EC2 instance maxes out CPU/network handling the traffic spike.
Everyone gets timeouts, including people not even looking at the viral post.

**Fix:** Bake an **AMI** with your app pre-installed, put an **ALB** in front, add an
**Auto Scaling Group** that adds instances as load increases.

**Components introduced:** AMI, ALB, ASG.
*(Module: 01-compute, 05-networking-edge — and this is your first "with/without load
balancer" experiment for real, not just as an isolated exercise)*

---

## Stage 4 — "Uploads take 8 seconds and people think the app is broken"

**Pain:** Every upload synchronously resizes the image into 3 thumbnail sizes inside
the request handler. Under load, this ties up app-server capacity and makes uploads
feel broken even though nothing actually failed.

**Fix:** Upload goes straight to S3. An **S3 event notification** fires into an
**SQS** queue. A **Lambda** worker picks up the message, generates thumbnails
asynchronously, writes results back to S3. The upload request now returns instantly.

**Components introduced:** SQS, Lambda, S3 event notifications.
*(Module: 06-messaging-events, 07-serverless)*

---

## Stage 5 — "The homepage feed is slow and RDS CPU is pegged"

**Pain:** Every page load re-runs the same "recent posts from people you follow"
query against RDS. At any real traffic volume this is almost entirely redundant
work — the underlying data barely changes between requests.

**Fix:** **ElastiCache (Redis)**, cache-aside pattern: check cache → miss → query
RDS → populate cache → next request hits cache. This is your with/without-cache
experiment from SKILL.md, except now it's solving a problem you actually caused
yourself, which is a much better way to learn it than a synthetic benchmark.

**Component introduced:** ElastiCache.

---

## Stage 6 — "International users say images load slowly"

**Pain:** Every image request round-trips to your one AWS region's S3 bucket,
regardless of where the user is.

**Fix:** **CloudFront** in front of S3 for images (edge caching, much shorter
round-trip for distant users). Add a real custom domain via **Route53** while you're
at it, instead of the default ALB/CloudFront hostname.

**Components introduced:** CloudFront, Route53.

---

## Stage 7 — "Why is our NAT Gateway bill $400 this month?"

**Pain:** Your app and Lambda workers, sitting in private subnets, route *all*
outbound traffic — including calls to S3 — through the NAT Gateway, which charges
per GB processed. S3 traffic is a huge chunk of that.

**Fix:** Add an **S3 Gateway VPC Endpoint**. Private-subnet resources now reach S3
directly over AWS's internal network — no NAT, no per-GB charge for that traffic,
and arguably more secure (never touches the public internet path at all).

**Component introduced:** VPC Endpoints.
*(This is the exact real-world "wait, I didn't need NAT for that" moment SKILL.md's
VPC Endpoints module describes — now you've hit it organically.)*

---

## Stage 8 — "Someone's scraping every image URL and someone else tried SQL injection in the comments box"

**Pain:** Public internet exposure attracts abuse once you have any real traffic.

**Fix:** **WAF** rules in front of CloudFront/ALB — rate-limit by IP, block common
injection patterns. **Shield Standard** is already on by default at this point, but
now you actually understand why it's there.

**Components introduced:** WAF, Shield.

---

## Stage 9 — "A second engineer joined and deploys are terrifying"

**Pain:** Deploying means SSHing into prod EC2 instances and running `git pull` +
restart. It's inconsistent, undocumented in practice, and port 22 open to the world
is now a legitimate security concern with two people needing access.

**Fix:** Containerize the app, push to **ECR**, run on **ECS Fargate** behind the
existing ALB (drop the EC2/ASG app tier entirely — Fargate manages the compute).
Replace ad-hoc SSH access with **SSM Session Manager** for the rare cases you still
need a shell — no open inbound port, full audit trail.

**Components introduced:** ECR, ECS (Fargate), SSM Session Manager.

---

## Stage 10 — "Deploys skip tests half the time because it depends who's doing it"

**Pain:** Manual `aws ecs update-service` from whoever's laptop is available. No
consistent test gate before prod.

**Fix:** **CodePipeline**: git push → **CodeBuild** (run tests, build image, push to
ECR) → **CodeDeploy** (roll out to ECS). While wiring this up, you also notice the DB
password has been sitting in a task definition environment variable this whole
time — move it to **Secrets Manager**, and pull non-sensitive config (a feature flag
for a new feed algorithm you're testing) from **SSM Parameter Store** instead of
hardcoding it.

**Components introduced:** CodePipeline/CodeBuild/CodeDeploy, Secrets Manager, SSM
Parameter Store.

---

## Stage 11 — "The app was slow for 20 minutes at 2am and nobody could tell why"

**Pain:** On-call has CPU graphs and nothing else. Was it the DB? The cache? A bad
deploy? A security group someone changed? No way to tell quickly.

**Fix:** Real **CloudWatch** dashboards and **Alarms** (not just default metrics —
alarms tied to actual SLOs). **CloudTrail** to answer "did someone change a security
group or IAM policy right before this started." **X-Ray** instrumented through the
request path (ALB → ECS → RDS/ElastiCache) to see exactly which hop was slow instead
of guessing.

**Components introduced:** CloudWatch Alarms (properly), CloudTrail, X-Ray.

---

## Stage 12 — "Postgres is falling over on one specific query pattern"

**Pain:** As DAUs grow, "get every like/view event for user X, right now" becomes a
hot, high-frequency, high-concurrency access pattern that's a bad fit for a
relational join-heavy schema — lock contention on that table is now affecting
unrelated queries.

**Fix:** Move *that specific access pattern* to **DynamoDB** — single-table design,
partition key = user ID. Everything else (users, captions, comments, relational
stuff) stays in RDS. This is the real lesson: polyglot persistence, not
"DynamoDB replaces RDS."

**Component introduced:** DynamoDB.

---

## Stage 13 — "Product wants a live trending feed, and the data team wants every view event too"

**Pain:** You'd need two independent consumers reading the same "image viewed"
events — analytics and trending — and a standard SQS queue only lets one logical
consumer group cleanly process each message once.

**Fix:** **Kinesis Data Streams** for view/interaction events. Both the trending
service and the analytics pipeline read the same stream independently, each at
their own pace, with replay if either falls behind.

**Component introduced:** Kinesis.

---

## Stage 14 — "Content moderation is now a legal requirement, and the pipeline is a mess of chained Lambdas"

**Pain:** New uploads must be scanned for policy violations before going public.
The pipeline is now: moderation scan → thumbnail generation → notify uploader →
possible human review queue. You'd been chaining these as Lambda-calls-Lambda in
code, and a failure partway through is nearly impossible to debug — you can't tell
which step failed or retry just that step.

**Fix:** **Step Functions** state machine: each step is explicit, retries and
branching (e.g., "flagged" vs "clean") are visual and inspectable, and a failed
execution shows you exactly where and why.

**Component introduced:** Step Functions.

---

## Stage 15 — "We need to reprocess 4 million existing images and workers keep redoing each other's work"

**Pain:** A one-time batch job regenerates thumbnails at a new size for every
existing image. You run it across multiple worker instances for speed, but they
have no shared state, so they keep clobbering each other's progress and
double-processing images.

**Fix:** **EFS** mounted by every worker instance as a shared coordination/checkpoint
directory. All workers see the same "already processed" state in real time — unlike
EBS, which only one instance can attach to.

**Component introduced:** EFS.

---

## Stage 16 — "A partner wants API access, and we're splitting into separate AWS accounts"

**Pain (two related ones at once, common as companies mature):**
1. A partner wants programmatic access to fetch a user's public images. Pointing
   them at your internal ALB directly gives you no per-partner API keys, rate
   limits, or usage tracking.
2. The company splits into separate dev/staging/prod AWS accounts to limit blast
   radius. A data analyst on a different team needs temporary read access to a prod
   S3 bucket for one report — not standing credentials.

**Fix:**
- **API Gateway** in front of a Lambda (or your ECS service) for the partner-facing
  API — API keys, per-partner throttling, usage plans.
- **STS AssumeRole** for the analyst's temporary cross-account access, scoped
  narrowly and time-limited.
- If a second internal service in a different VPC now needs to talk to Pixly's
  private resources, **VPC Peering** connects the two (and if a third VPC joins
  later, that's your natural trigger to graduate to **Transit Gateway**, since
  peering isn't transitive).

**Components introduced:** API Gateway, STS/AssumeRole, VPC Peering (→ Transit
Gateway).

---

## Stage 17 — "An entire AWS Availability Zone went down and so did we"

**Pain:** Everything — RDS, ECS tasks, cache nodes — happened to be concentrated in
one AZ. That AZ has an outage. You're fully down instead of degraded.

**Fix:** RDS Multi-AZ, ECS tasks and ElastiCache spread across ≥2 AZs behind the
ALB. Actually simulate the outage (stop everything in one AZ) and time the recovery
— don't just configure it and assume it works.

**Component reinforced:** Multi-AZ patterns (this is SKILL.md's
`10-architectures/multi-az-failover/` module, now with real stakes attached).

---

## Stage 18 — "Should we move to Kubernetes?" (a choice, not a forced pain point)

Unlike every stage above, this one isn't caused by ECS breaking — it's a strategic
fork some real companies hit: a platform team wants workload portability across
clouds, or existing Kubernetes expertise on a newly-merged team makes standardizing
worthwhile. Migrate the ECS service to **EKS** as a deliberate architecture decision,
and be honest with yourself in your notes about what you gained (portability,
K8s ecosystem tooling) versus what you gave up (ECS's simplicity, tighter native AWS
integration).

**Component introduced:** EKS.
*(Flagging this one as a choice rather than a pain point is itself the lesson —
not every real architecture decision is forced by a failure.)*

---

## What this leaves out on purpose

KMS and NACLs aren't in a dedicated stage above because they're better learned as
*retrofits*, which is realistic: e.g., "Stage 16.5 — first EU enterprise customer's
contract requires documented encryption-at-rest key management" → go back and add
**KMS** customer-managed keys to RDS/S3/EBS retroactively. Compliance requirements
showing up after the fact and forcing you to bolt security onto existing
infrastructure is extremely true to how this happens in real companies — don't
smooth that over by pre-planning it into the main timeline.

---

## How to use this alongside SKILL.md

For each stage: build the fix, then go read that component's full entry in
`SKILL.md` (dependencies, Floci gotchas, "definition of done" questions) and make
sure you can answer all four questions there using *this specific stage's pain* as
your example — not a generic textbook answer.
