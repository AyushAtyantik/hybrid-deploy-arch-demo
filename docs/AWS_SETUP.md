# AWS setup — runbook

Zero to deployed, in order. Roughly **40 minutes** of console work, most of it while RDS
provisions in the background.

Everything here is done **once**. After that the application ships itself: see
[DEPLOYMENT.md](DEPLOYMENT.md). For *why* the architecture is shaped this way, see
[ARCHITECTURE.md](ARCHITECTURE.md).

Region used throughout: **`ap-south-1`**. If you change it, change `AWS_REGION` in
`.github/workflows/deploy.yml` to match.

## Target state

```mermaid
flowchart TB
    Browser(["Browser"])
    Worker["Cloudflare Worker<br/>static assets + /api/* proxy"]
    IGW{{"Internet Gateway"}}

    subgraph VPC["VPC · 10.0.0.0/16"]
        direction TB
        subgraph PUB["Public subnets · 2 AZs"]
            direction TB
            ALB["Application Load Balancer<br/>HTTP :80"]
            TG["Target group<br/>:3000 · health /health"]
            ASG["Auto Scaling Group<br/>min 2 · max 6 · t3.micro"]
        end
        subgraph PRIV["Private subnets · 2 AZs"]
            RDS[("RDS MySQL<br/>db.t4g.micro · no public access")]
        end
    end

    Browser -- HTTPS --> Worker
    Worker -- "HTTP, server-side" --> IGW
    IGW --> ALB --> TG --> ASG
    ASG -- ":3306" --> RDS
```

## Three strings that must match

Most failures in this runbook are a mismatch between the console and the repo. Check these
before you start and again at the end.

| Value | Set in the console | Referenced in the repo |
|---|---|---|
| `campuswall-asg` | Auto Scaling Group name | `ASG_NAME` in `.github/workflows/deploy.yml` |
| `ap-south-1` | Region you build in | `AWS_REGION` in the same file |
| ALB DNS name | Output of step 7 | `ALB_HOST` in `apps/web/wrangler.toml` |

---

# Part 1 — Before AWS

## Step 1. Publish the repository

Instances clone the repo at boot, with no credentials, so it **must be public**.

1. Push this repository to GitHub.
2. **Settings → General → Danger Zone → Change visibility → Public.**
3. Confirm `pnpm-lock.yaml` is committed — CI and the EC2 bootstrap both use
   `--frozen-lockfile` and fail without it.

Then set your repo URL in `infra/user-data.sh` and commit it:

```bash
REPO="https://github.com/<YOUR_GITHUB_USER>/hybrid-deploy-arch-demo"
```

> ⚠️ **That is the only placeholder in that file you may commit.** `<RDS_ENDPOINT>` and
> `<RDS_PASSWORD>` are filled in **inside the Launch Template's user-data box** in step 8,
> never in the repository. The repo is public.

## Step 2. Cloudflare credentials

1. **dash.cloudflare.com → Manage Account → Account API Tokens → Create Token.**
2. Use the **Edit Cloudflare Workers** template. Create it and copy the token once — it is
   never shown again.
3. Copy your **Account ID** from the right-hand sidebar of the dashboard home.

---

# Part 2 — IAM

Do this before building anything. The instance role is a dropdown in step 8, and a missing
one causes a silent failure much later.

## Step 3. EC2 instance role

**IAM → Roles → Create role → AWS service → EC2 → Next.**

1. Attach **`AmazonSSMManagedInstanceCore`**.
2. Name it **`campuswall-ec2-role`**. Create.

The SSM Agent ships with Amazon Linux 2023, so nothing else is needed. This role gives you:

- **SSM Run Command** — how deployments reach the fleet
- **Session Manager** — a browser shell into any instance, no key pair, no port 22
- Optionally SSM Parameter Store, to get the database password out of user-data later

Without it, instances launch and run fine but **deployments report success and change
nothing.**

## Step 4. GitHub OIDC provider

So Actions can assume a role without any stored AWS keys.

**IAM → Identity providers → Add provider → OpenID Connect.**

- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

## Step 5. Deploy role

**IAM → Roles → Create role → Custom trust policy.** Paste, substituting your account ID,
GitHub username and repo name:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike":   { "token.actions.githubusercontent.com:sub": [
        "repo:<YOUR_GITHUB_USER>/hybrid-deploy-arch-demo:ref:refs/heads/main",
        "repo:<YOUR_GITHUB_USER>@*/hybrid-deploy-arch-demo@*:ref:refs/heads/main"
      ] }
    }
  }]
}
```

> ⚠️ **The `sub` condition is the line that matters.** Without it, *any* GitHub repository
> in the world can assume this role. It is also **case-sensitive** — IAM compares literally,
> so `repo:myname/...` will not match a repo owned by `MyName`.

### The `sub` claim has two possible formats

This is the single most likely thing to go wrong, and the error message gives you nothing.

GitHub emits **one of two** subject formats, depending on the account:

```
repo:OWNER/REPO:ref:refs/heads/main                      # name-based (older)
repo:OWNER@1234567/REPO@89012345:ref:refs/heads/main     # with immutable numeric IDs
```

The second form pins the claim to numeric owner and repository IDs so it survives renames.
If your account emits it and your policy expects the first, every assume-role fails with a
flat `Not authorized to perform sts:AssumeRoleWithWebIdentity` — no hint that the `sub` is
the problem.

**Do not guess which one you get.** The `${{ github.repository }}` context does *not* tell
you: it always prints the name-based form. Accept both instead — `StringLike` takes an
array, and any match wins:

```json
"StringLike": {
  "token.actions.githubusercontent.com:sub": [
    "repo:<OWNER>/<REPO>:ref:refs/heads/main",
    "repo:<OWNER>@*/<REPO>@*:ref:refs/heads/main"
  ]
}
```

Once you know your real claim you can pin it exactly, which is marginally stronger:
**CloudTrail → Event history → Event name `AssumeRoleWithWebIdentity`** → open the event →
`userIdentity.userName` is the literal `sub` GitHub sent. Copy it verbatim.

A run on any other branch, a tag, or a pull request emits a different `sub` and is refused
by this policy. That is intended.

Attach an inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["ssm:SendCommand", "ssm:ListCommandInvocations", "ssm:GetCommandInvocation"],
    "Resource": "*"
  }]
}
```

Name the role **`campuswall-gha-deploy`** and copy its ARN.

## Step 6. GitHub secrets

**Repo → Settings → Secrets and variables → Actions → New repository secret.**

| Secret | Value |
|---|---|
| `AWS_DEPLOY_ROLE_ARN` | `arn:aws:iam::<ACCOUNT_ID>:role/campuswall-gha-deploy` |
| `CLOUDFLARE_API_TOKEN` | From step 2 |
| `CLOUDFLARE_ACCOUNT_ID` | From step 2 |

---

# Part 3 — Network

## Step 7. VPC

**VPC → Create VPC → "VPC and more"** (the wizard — it builds subnets, route tables and the
internet gateway in one go).

| Setting | Value |
|---|---|
| Name tag auto-generation | `campuswall` |
| IPv4 CIDR | `10.0.0.0/16` |
| IPv6 | No |
| Availability Zones | **2** |
| Public subnets | **2** |
| Private subnets | **2** |
| **NAT gateways** | **None** |
| VPC endpoints | None |
| DNS hostnames / resolution | **Both enabled** |

NAT gateways cost ~$32/month and add several minutes. The app tier lives in the public
subnets instead — a deliberate trade-off explained in
[ARCHITECTURE.md](ARCHITECTURE.md#why-the-app-tier-is-in-public-subnets).

> ⚠️ **Now turn on auto-assign public IPv4 for both public subnets.**
>
> **VPC → Subnets →** select `…-public1-…` **→ Actions → Edit subnet settings → ✓ Enable
> auto-assign public IPv4 address.** Repeat for `…-public2-…`.
>
> With no NAT gateway, a public IP is the *only* route these instances have to the
> internet. Without one they still launch and still show as `Running`, but nothing works:
> user-data can't `git clone`, so the app is never installed, and the SSM Agent can't
> register, so deployments match zero instances. The instance summary shows
> `Public IPv4 address: –` and `Managed: false`.

Take a moment on the **resource map** the wizard draws. It shows the subnets, both route
tables and the internet gateway, and it is the clearest picture AWS gives you for free.

## Step 8. Security groups

**EC2 → Security Groups → Create security group.** Create all three in this order, in the
`campuswall-vpc`. Each one's rule points at **the group before it**, never a CIDR.

```mermaid
flowchart LR
    Net(["Internet"]) -- ":80" --> alb["alb-sg"]
    alb -- ":3000" --> ec2["ec2-sg"]
    ec2 -- ":3306" --> rds["rds-sg"]
```

| # | Name | Inbound rule | Source |
|---|---|---|---|
| 1 | `alb-sg` | HTTP `80` | `0.0.0.0/0` |
| 2 | `ec2-sg` | Custom TCP `3000` | **`alb-sg`** |
| 3 | `rds-sg` | MySQL/Aurora `3306` | **`ec2-sg`** |

> ⚠️ **These are INBOUND rules. All three of them.**
>
> Security groups are **stateful** — the reply to an accepted connection is allowed
> automatically, so the rule only ever goes on the side *receiving* the connection. Adding
> "MySQL/Aurora 3306 → ec2-sg" to `rds-sg`'s **outbound** tab instead is an easy click to get
> wrong, and it does nothing: it governs connections the database itself initiates.
>
> The symptom is distinctive — `/health` returns `{"db": false}` after **~5 seconds**
> (`connectTimeout`), because packets are dropped with no reply. Wrong credentials or a
> missing database fail in *milliseconds* instead.

Leave outbound as the default (all traffic). Add `443` to `alb-sg` only if you later attach
an ACM certificate; this setup uses plain HTTP because the Worker calls the ALB server-side.

**There is no port 22 anywhere.** Shell access is Session Manager.

---

# Part 4 — Database

Start this now. It takes ~8 minutes, and you will build the load balancer while it works.

## Step 9. DB subnet group

**RDS → Subnet groups → Create DB subnet group.**

| Field | Value |
|---|---|
| Name | `campuswall-db-subnets` |
| Description | `private subnets for campuswall` |
| VPC | `campuswall-vpc` |
| Availability Zones | Both |
| Subnets | **The two private subnets only** |

Check the subnet IDs against the VPC resource map. The private subnets are the ones whose
route table has **no** `0.0.0.0/0 → igw` entry. Adding a public subnet here is what later
lets someone flip the database to internet-facing.

## Step 10. Create the database

**RDS → Databases → Create database.** The console asks for a lot here; most of it can stay
at its default. The sections below appear in the order the console shows them, and flag the
handful that actually matter.

### Engine options

> ⚠️ **Choose `MySQL`, not `Aurora (MySQL Compatible)`.**
>
> Aurora is the **first tile and selected by default**, and it is easy to skip past. Aurora
> is a different product with different pricing: it has **no `micro` instance class** (the
> smallest burstable option offered is `db.t4g.medium`, 2 vCPU / 4 GiB), and it bills
> storage and I/O separately. None of the cost figures in this runbook apply to it.
>
> **You are on Aurora if you can see any of these:** *Cluster scalability type*, *Cluster
> storage configuration*, *DB cluster identifier*, *Create an Aurora Replica or Reader
> node*, *Read replica write forwarding*, or an engine version reading *"Aurora MySQL 8.4.7
> (compatible with MySQL 8.4.7)"*. Go back and pick the plain **MySQL** tile.

| Field | Value |
|---|---|
| Engine type | **MySQL** |

### Choose a database creation method

**Full configuration.** (This used to be labelled *Standard create*.) The alternative,
*Easy create*, hides the VPC, subnet group and security group settings — exactly the ones
this architecture depends on.

### Templates

Pick **Dev/Test**. *Production* pre-selects Multi-AZ, Provisioned IOPS and deletion
protection, all of which you would then have to undo.

**On the Free tier template:** AWS replaced the 12-month Free Tier on **15 July 2025**.
Accounts created after that date get a **6-month free plan plus up to $200 in credits**, and
the *Free tier* template no longer appears — you will see only **Production** and
**Dev/Test**. That is expected. `db.t4g.micro` is still the right class; just set the values
explicitly below.

### Availability and durability

**Single DB instance.** The Multi-AZ options double or triple the cost. Multi-AZ is the
correct production answer and worth knowing about; it is not worth paying for here.

### Settings

| Field | Value |
|---|---|
| Engine version | **MySQL 8.4.x** — take the newest minor offered |
| **Enable RDS Extended Support** | **Leave unchecked** |
| DB instance identifier | `campuswall-db` |
| Master username | `admin` |

> ⚠️ **Do not pick MySQL 8.0.** It left RDS standard support on **31 July 2026** and now runs
> only under **RDS Extended Support**, which is a paid offering. 8.4 is in standard support
> until 31 July 2029 and matches the `mysql:8.4` image in `docker-compose.yml`.
>
> The **Enable RDS Extended Support** checkbox sits directly under *Engine version*. Ticking
> it consents to charges once your major version passes its end-of-standard-support date.
> On 8.4 that is years away, but leave it off so a forgotten stack fails loudly instead of
> billing quietly.

### Credentials Settings

> ⚠️ **Change *Credentials management* to "Self managed".**
>
> **"Managed in AWS Secrets Manager" is selected by default.** It is the better practice and
> the console recommends it — but the instance bootstrap reads a plaintext password from
> user data, so you need one you can copy. Secrets Manager also carries its own charges.
>
> Choose **Self managed**, clear **Auto generate a password**, and set a password you
> control. Save it now: you need it in step 13, and it cannot be retrieved later — only
> reset by modifying the instance.

Under **Additional credentials settings** you will find **Database authentication**. Leave
both **IAM database authentication** and **Kerberos authentication** unchecked; password
authentication is always active and is what the app uses.

### Instance configuration

Select **Burstable classes (includes t classes)**, then **`db.t4g.micro`**.

If the smallest class you are offered is `db.t4g.medium`, you are still on Aurora — go back
to *Engine options*.

### Storage

| Field | Value |
|---|---|
| Storage type | **General Purpose SSD (gp3)** |
| Allocated storage | **20** GiB |
| **Storage autoscaling** | **Uncheck "Enable storage autoscaling"** |

Autoscaling is on by default with a **1,000 GiB** maximum. Nothing here will grow, and an
unbounded ceiling on a throwaway stack is a bad habit.

### Connectivity

| Field | Value |
|---|---|
| Compute resource | **Don't connect to an EC2 compute resource** |
| Network type | **IPv4** |
| Virtual private cloud (VPC) | **`campuswall-vpc`** — not `Default VPC` |
| DB subnet group | **`campuswall-db-subnets`** — not `default` |
| **Public access** | **No** |
| VPC security group (firewall) | **Choose existing** → `rds-sg`, and remove the `default` chip |
| Availability Zone | No preference |
| Certificate authority | Leave the default (`rds-ca-rsa2048-g1`) |

Both the **VPC** and **DB subnet group** fields default to `Default VPC` / `default`. If you
leave them, the database lands outside the network you built in steps 7–9 and nothing will
be able to reach it. **You cannot change a database's VPC after creation** — the console
says so inline.

> ⚠️ **"Connect to an EC2 compute resource" looks helpful and is the wrong choice.** It makes
> RDS invent its own subnet group and a pair of `rds-ec2-*` security groups, silently
> bypassing the `alb-sg → ec2-sg → rds-sg` chain from step 8. Instances are launched later by
> an Auto Scaling group anyway, so there is nothing to attach to yet.

The **Database port** (`3306`) lives under *Connectivity → Additional configuration*.

### Monitoring

| Field | Value |
|---|---|
| Database Insights | **Standard** — detailed metrics, free at 7-day retention |
| Enable collecting detailed per-query metrics | Fine to leave checked |
| Retention period | **7 days (free)** |

Then expand **Additional monitoring settings**:

> ⚠️ **"Enable Enhanced Monitoring" is checked by default — uncheck it.**
>
> It publishes OS metrics to CloudWatch Logs at 60-second granularity and is billed
> separately. Leaving it on also creates an IAM role called `rds-monitoring-role` that you
> then have to remember at teardown. You do not need per-process CPU metrics for this.

Leave every **Log exports** checkbox (Audit, Error, General, iam-db-auth-error, instance,
Slow query) unticked — each one is a billed CloudWatch Logs stream.

### Additional configuration

Expand it. This is where the field that breaks everything lives.

> ⚠️ **Initial database name: `campuswall`**
>
> The single most commonly missed field in this runbook. The application connects to a
> database called `campuswall` and creates its **tables** on boot — but it does not create
> the **database**. Leave this blank and RDS creates no database at all: every instance sits
> at `/health` → `503 degraded` indefinitely, with nothing in the logs pointing at the cause.

| Field | Value |
|---|---|
| Initial database name | **`campuswall`** |
| DB parameter group | `default.mysql8.4` |
| Option group | Default |
| Automated backups | **Uncheck "Enable automated backups"** — nothing here is worth keeping |
| Encryption | Leave **enabled** (default `aws/rds` key, no extra cost) |
| Auto minor version upgrade | Leave enabled |
| Maintenance window | No preference |
| **Deletion protection** | **Leave unchecked** — you want teardown to be easy |

Choose **Create database**, then move straight to Part 5 while it provisions (~8 minutes).
Once it reaches **Available**, open it and copy the **endpoint** — you need it in step 13.

---

# Part 5 — Load balancer

## Step 11. Target group

**EC2 → Target Groups → Create target group → Instances.**

| Setting | Value |
|---|---|
| Name | `campuswall-tg` |
| Protocol / port | HTTP **3000** |
| VPC | `campuswall-vpc` |
| Protocol version | HTTP1 |
| Health check protocol / path | HTTP **`/health`** |

Open **Advanced health check settings** — the defaults are both too slow to watch and too
twitchy under load:

| Setting | Value | Why |
|---|---|---|
| Healthy threshold | **2** | ~20s to join the pool |
| Unhealthy threshold | **5** | ~50s of failures — one slow response can't evict |
| Timeout | **5s** | Survives a loaded event loop |
| Interval | **10s** | Fast enough to watch state change |
| Success codes | `200` | |

Do **not** register any targets — the ASG does that. Create it, then open the group's
**Attributes** tab and set **Deregistration delay to 30 seconds** (the default 300 makes
draining tediously slow to demonstrate). Leave **stickiness off**; turning it on pins each
client to one instance and hides load balancing entirely.

## Step 12. Application Load Balancer

**EC2 → Load Balancers → Create → Application Load Balancer.**

| Setting | Value |
|---|---|
| Name | `campuswall-alb` |
| Scheme | **Internet-facing** |
| IP address type | IPv4 |
| VPC | `campuswall-vpc` |
| Mappings | Both AZs, selecting the **public** subnet in each |
| Security group | `alb-sg` (remove `default`) |
| Listener | **HTTP : 80** → forward to `campuswall-tg` |

**Copy the DNS name** from the load balancer's detail page —
`campuswall-alb-123456789.ap-south-1.elb.amazonaws.com`. You need it in step 16.

---

# Part 6 — Compute

By now RDS should be **Available**. Open it and copy the **endpoint**.

## Step 13. Launch template

**EC2 → Launch Templates → Create launch template.**

| Setting | Value |
|---|---|
| Name | `campuswall-lt` |
| AMI | **Amazon Linux 2023**, x86_64 |
| Instance type | `t3.micro` |
| Key pair | **Do not include** (Session Manager instead) |
| Subnet | **Don't include in template** (the ASG picks) |
| Security group | `ec2-sg` |
| Auto-assign public IP | **Enable**, or leave it to the subnet default from step 7 |

If the launch template defines a network interface, its *Auto-assign public IP* setting
**overrides** the subnet default. Either set it to `Enable` here, or don't define a network
interface at all and let the subnet decide.

Then expand **Advanced details**:

| Setting | Value | Why |
|---|---|---|
| IAM instance profile | **`campuswall-ec2-role`** | Without it, deploys silently do nothing |
| Detailed CloudWatch monitoring | **Enable** | 1-minute metrics; the scaling alarm needs them |
| Credit specification | **Unlimited** | See below |
| Metadata version | **V2 only** | The app uses IMDSv2 |

**Credit specification matters.** `t3` is burstable: sustained CPU spends credits, and when
they run out the instance is throttled to **10% of a vCPU** — the app crawls and does not
recover. Under `unlimited` you pay a few cents for surplus credits instead.

Finally, **User data**: paste the contents of `infra/user-data.sh` and edit the two
placeholders **here, in the console**:

```bash
DB_HOST=<RDS_ENDPOINT>       # the endpoint you just copied, no port
DB_PASS=<RDS_PASSWORD>       # the master password from step 10
```

> ⚠️ **Do not commit those two values.** The repository is public. The `REPO=` line should
> already be correct from step 1.

## Step 14. Auto Scaling Group

**EC2 → Auto Scaling Groups → Create Auto Scaling group.**

| Setting | Value |
|---|---|
| Name | **`campuswall-asg`** — must match `ASG_NAME` in the workflow |
| Launch template | `campuswall-lt` |
| VPC / subnets | `campuswall-vpc`, both **public** subnets |
| Load balancing | Attach to an existing load balancer → `campuswall-tg` |
| **Health check type** | **EC2 only** — leave ELB health checks **off** |
| Health check grace period | `300` seconds |
| Desired / Min / Max | **2 / 2 / 6** |
| Scaling policies | None (added next) |

**Health check type `EC2`, not `ELB`,** while you are experimenting. With `ELB`, a target
that flaps unhealthy under load gets *terminated* by the ASG; with `EC2` the ALB simply
stops routing to it and it recovers on its own. `ELB` is the right production choice once
the workload is stable.

Within ~3 minutes the target group should show two **healthy** targets. If not, jump to
[Troubleshooting](#troubleshooting).

## Step 15. Scaling policy

Two pieces: an alarm, then a policy that reacts to it.

**CloudWatch → Alarms → Create alarm → Select metric → EC2 → By Auto Scaling Group →
`campuswall-asg` → `CPUUtilization`.**

| Setting | Value |
|---|---|
| Statistic | Average |
| Period | **1 minute** |
| Threshold type | Static, **Greater than 40** |
| Datapoints to alarm | **1 of 1** |
| Missing data | Treat as missing |
| Notification | Remove it (no SNS needed) |
| Name | `campuswall-cpu-high` |

**Use step scaling, not target tracking.** Target tracking builds its own alarm requiring
three consecutive datapoints — roughly three minutes before it even fires.

Then **EC2 → Auto Scaling Groups → `campuswall-asg` → Automatic scaling → Create dynamic
scaling policy**:

| Setting | Value |
|---|---|
| Policy type | **Step scaling** |
| Name | `campuswall-scale-out` |
| CloudWatch alarm | `campuswall-cpu-high` |
| Action | **Add `2` capacity units** when `40 <= CPUUtilization < +infinity` |
| Instance warmup | `60` seconds |

Optionally add a scale-**in** policy on a second alarm (`CPUUtilization < 15` for 3
datapoints, remove 1 instance). Scale-in is deliberately slower than scale-out.

---

# Part 7 — Deploy

## Step 16. Point the Worker at the ALB

Edit `apps/web/wrangler.toml` — host name only, no `http://`, no trailing slash:

```toml
[vars]
ALB_HOST    = "campuswall-alb-123456789.ap-south-1.elb.amazonaws.com"
SHOW_STRESS = "false"
```

Commit and **push to `main`**. That is the deploy. Watch the **Actions** tab: `deploy-web`
takes ~30s, `deploy-api` ~90s.

There is nothing to run locally — no `wrangler`, no `aws`, no `ssh`. Configuration lives in
the repo, so changing it is a commit. See [DEPLOYMENT.md](DEPLOYMENT.md).

## Step 17. Verify

Against the ALB directly:

```bash
curl -s http://<alb-dns>/health | jq
# → {"status":"ok","db":true,"served_by":"i-0…","az":"ap-south-1a","uptime_s":…}

for i in $(seq 1 20); do curl -s http://<alb-dns>/api/whoami | jq -r .served_by; done
# → two instance ids, alternating
```

`"db": true` confirms the security group chain and the initial database name are right.

Then open your `*.workers.dev` URL: the client view at `/` and the display view at `/wall`.

## Step 18. Reveal the load generator

Set `SHOW_STRESS = "true"` in `wrangler.toml`, commit, push. About 40 seconds later the
button appears for everyone. This is also how you turn it off afterwards.

**Leave it `"false"` except when you are actively demonstrating.** `/api/stress` is an
unauthenticated endpoint that burns CPU on request.

---

# Exercising the setup

## Watching it scale

Normal traffic will not move CPU on a `t3.micro` — a few small inserts and an indexed select
is single-digit percent. `/api/stress` exists to create real load.

```
Capacity:  2 instances × 2 vCPU  = 4 vCPU-seconds per second
Load:      N requests/sec × B seconds of burn
```

| Goal | Required |
|---|---|
| Hold CPU above the 40% threshold | `N × B ≥ 1.6` |
| Peg CPU at ~100% | `N × B ≥ 4.0` |

At `BURN_MS = "500"` (set in `apps/web/wrangler.toml`), about **3.2 requests/second** clears
the threshold and **8/second** saturates. Overshooting is harmless — CPU pegs and the queue grows, which is the condition
scaling exists for. Undershooting means the alarm never fires.

Generate it deterministically:

```bash
npx autocannon -c 8 -d 180 -m POST "http://<alb-dns>/api/stress?ms=500"
```

Expect **3–5 minutes** end to end:

| Phase | Duration |
|---|---|
| Metric publish + alarm evaluation | 60–120s |
| EC2 launch to `running` | 30–45s |
| user-data: install, clone, build | 60–90s |
| Health check passes (10s × 2) | ~20s |

To make it faster, bake a **custom AMI** with Node and dependencies pre-installed (cuts
user-data to ~15s, total to ~2 minutes), or use an **ASG warm pool** (~30 seconds).

## What to watch

Build a CloudWatch dashboard with four widgets: `CPUUtilization`, `TargetResponseTime`,
`RequestCount`, `HealthyHostCount`. `TargetResponseTime` rising and then recovering as new
instances join is the clearest single picture of autoscaling working.

Also keep **Target groups → `campuswall-tg` → Targets** open: `initial` → `healthy` →
`draining` → gone, live.

**ALB access logs require an S3 bucket** and are deliberately not used here. CloudWatch
metrics cover everything needed.

## Failure behaviour worth trying

| Action | Expected |
|---|---|
| Terminate an instance (EC2 console) | ALB drains it, ASG launches a replacement, no downtime |
| Delete the `0.0.0.0/0 → igw` route from the public route table | Everything stops. Restore it and it returns |
| `mysql -h <rds-endpoint>` from your laptop | Hangs — it is in a private subnet |
| Same command from a Session Manager shell | Connects immediately |

---

# Troubleshooting

| Symptom | Cause |
|---|---|
| Targets stuck `unhealthy` | Session Manager in, then `cat /var/log/user-data.log`, `systemctl status campuswall`, `journalctl -u campuswall -n 50` |
| `/health` returns `degraded`, `"db": false` **after ~5 seconds** | Packets are being dropped — a network problem, not a credentials one. Check `rds-sg` has an **inbound** rule for 3306 from `ec2-sg` (not an *outbound* rule), that RDS is attached to `rds-sg`, and that the instances are in `ec2-sg`. A security group recreated later gets a **new id**, leaving the old rule pointing at a `sg-…` that no longer exists |
| `/health` returns `degraded`, `"db": false` **immediately** | Credentials or database name — **Initial database name** not set to `campuswall` at step 10, or `DB_HOST`/`DB_PASS` wrong in user-data. Read the exact MySQL error with `journalctl -u campuswall` |
| Console shows *Cluster scalability type* or *DB cluster identifier* | You selected **Aurora**, not MySQL. Aurora has no `micro` class and prices differently — start step 10 again |
| Smallest instance class offered is `db.t4g.medium` | Same cause: you are on Aurora |
| No password to put in user-data | *Credentials management* was left on **Managed in AWS Secrets Manager**. Modify the instance and set a self-managed password |
| Instances can reach the internet but not the database | The database was created in `Default VPC` — the VPC can't be changed after creation, so delete and recreate it |
| `Connect` button greyed out in EC2 console | Instance profile missing from the launch template — instances already running need replacing |
| Deploy is green but nothing changed | ASG name ≠ `ASG_NAME`, or no instance profile. SSM matched zero targets and still succeeded |
| `deploy-api` reports **SSM matched 0 instances**, and the instance shows `Managed: false` | The SSM Agent never registered. Check **Public IPv4 address** on the instance — if it is `–`, the instance has no internet route (no NAT by design), so the agent cannot reach the SSM endpoints. Fix auto-assign public IPv4 (step 7) and **replace the instances** |
| Instances are `Running` but targets never go healthy, and `/var/log/user-data.log` is missing or stops at `git clone` | Same root cause: no public IP, so no internet |
| `deploy-api` fails at `configure-aws-credentials` with `Not authorized to perform sts:AssumeRoleWithWebIdentity` | The trust policy refused the token. Most often the **`sub` format** — GitHub may send `repo:OWNER@123/REPO@456:...` rather than `repo:OWNER/REPO:...`. Check CloudTrail (below) before anything else. Then: casing, wrong owner/repo, a branch other than `main`, a provider that was never created, or a bad `AWS_DEPLOY_ROLE_ARN` |
| Same, and you want the actual claim | CloudTrail → Event history → Event name `AssumeRoleWithWebIdentity`. `userIdentity.userName` in the failed event **is** the `sub` GitHub sent. This is the only reliable way to see it — the workflow context cannot tell you |
| Worker returns `502 origin unreachable` | `ALB_HOST` wrong, has a scheme or trailing slash, or `alb-sg` isn't open on 80 |
| Everything slows down after a few minutes of load | `t3` credits exhausted — credit specification must be `unlimited` |
| Instances cycle endlessly under load | Health check timeout too low, or ASG health check type set to `ELB` |
| Only one instance ever answers | Target group stickiness is on |

**Session Manager:** EC2 → Instances → select → **Connect** → *Session Manager* tab. No key
pair, no port 22, no bastion.

---

# Teardown

Delete in this order or dependencies block you:

1. **Auto Scaling Group** — set desired/min to `0`, wait for instances to terminate, delete
2. **Load balancer**, then the **target group**
3. **RDS instance** — skip the final snapshot for a throwaway stack
4. **Launch template**
5. **CloudWatch alarms**
6. **VPC** — removes subnets, route tables and the internet gateway
7. Optionally the IAM roles and the OIDC provider

## Cost

| Resource | Rate |
|---|---|
| ALB | ~$0.0225/hr + LCU |
| 2–6 × t3.micro | ~$0.0104/hr each |
| RDS db.t4g.micro | ~$0.016/hr |
| NAT Gateway | **$0 — avoided** |
| Cloudflare Workers | Free tier |

About **$0.06/hour** running, so well under **$0.20** for a two-hour session. Left up
continuously it is **~$45/month**, dominated by the ALB and RDS. **Set a reminder to tear it
down**, and check the Billing dashboard afterwards.
