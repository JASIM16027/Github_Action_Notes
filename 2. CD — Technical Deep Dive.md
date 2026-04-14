# CI/CD — Technical Deep Dive

### Real Example: Application Flight Engine Pipeline

---

## প্রথমে বুঝি — কী হয় যখন `git push` করো?

Application-এর flight-engine repo-তে কেউ push করলে ঠিক এটা হয়:

```
Developer PC              GitHub Server                GitHub Actions Runner
────────────              ─────────────                ────────────────────
git push ────────────────► code receive
                           event fire ────────────────► fresh Ubuntu VM তৈরি
                                                        কোড clone
                                                        AWS credentials set
                                                        ECR-এ login
                                                        docker build (b2c + b2b)
                                                        docker push → ECR
                                                        GitOps repo update
                                                        Kustomize → K8s জানলো
                                                        Slack notification গেলো
```

GitHub internally একটা **webhook event** fire করে। সেই event GitHub Actions কে trigger করে। Actions একটা **fresh virtual machine** (runner) তৈরি করে এবং `.github/workflows/` এর `.yml` ফাইল পড়ে কাজ শুরু করে।

---

## Runner কী জিনিস?

Runner হলো GitHub এর একটা **virtual machine** — যেখানে তোমার সব কাজ চলে।

```
GitHub Hosted Runner (এই pipeline-এ যেটা use হচ্ছে):
├── OS: Ubuntu Latest
├── RAM: 7 GB
├── CPU: 2 core
├── Disk: 14 GB
├── Pre-installed: Docker, git, AWS CLI, Node, Python...
└── প্রতিটা job এ FRESH — আগের কিছু থাকে না
```

```yaml
# Application pipeline-এ:
runs-on: ubuntu-latest   # GitHub এর free VM
```

প্রতিটা job শেষে VM destroy হয়ে যায়। পরের job এ একদম নতুন VM।

---

## Webhook → Event → Trigger — ভেতরে কী হয়?

```
Developer: git push origin develop
        ↓
GitHub এর git server কোড receive করলো
        ↓
GitHub internally event generate করলো:
{
  "event": "push",
  "ref": "refs/heads/develop",
  "commits": [...],
  "pusher": { "name": "developer-name" },
  "repository": { "name": "flight-engine" }
}
        ↓
এই event workflow এর "on:" section এর সাথে match করলো:

on:
  push:
    branches:
      - develop    ✅ match!
      - master
      - staging
        ↓
Runner queue তে job add হলো
        ↓
Available runner এ job চালু হলো
```

---

## একটা Step এ আসলে কী হয়?

```yaml
# Application pipeline এর একটা step:
- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@v1
```

এই একটা step এ যা হয়:

```
Runner এর shell (bash) এ:
  ┌──────────────────────────────────────────────────┐
  │  aws ecr get-login-password --region ap-south.. │
  │  | docker login --username AWS               │
  │    --password-stdin 123456.dkr.ecr.aws       │
  │                                              │
  │  > Login Succeeded                           │
  │  > exit code: 0                              │
  └──────────────────────────────────────────────┘

exit code 0   → success → পরের step চলবে
exit code 1+  → failure → job বন্ধ, Slack-এ error যাবে
```

---

## actions/checkout — ভেতরে কী করে?

```yaml
# Pipeline এর প্রথম step:
- name: Checkout
  uses: actions/checkout@v1
```

এটা আসলে একটা Node.js script যা এই কাজ করে:

```bash
# internally যা হয়:
git clone https://github.com/Applicationnet/flight-engine.git .
git checkout a3f8c12def456...  # যে commit push হয়েছে
```

Runner এ flight-engine এর পুরো কোড নামিয়ে আনে।
এটা ছাড়া `Dockerfile.development` বা অন্য কোনো file পাবে না।

---

## Context & Expression — Dynamic Value কীভাবে আসে?

```yaml
# Pipeline এ এভাবে use হচ্ছে:
docker build -t $ECR_REPO:${{ github.sha }} -f Dockerfile.${ENVIRONMENT} .
```

`${{ }}` হলো **expression syntax** — runtime এ replace হয়:

```
github.sha        → a1b2c3d4e5f6...  (commit hash, image tag হিসেবে use)
github.ref        → refs/heads/develop (কোন branch জানতে)
github.actor      → যে developer push করেছে
github.workspace  → /home/runner/work/flight-engine/
                    (কোড কোথায় আছে runner এ)

# Pipeline এ branch detect করতে:
if [ "${GITHUB_REF}" == "refs/heads/develop" ]; then
    ENVIRONMENT="development"
fi
```

---

## Secrets — Technically কীভাবে কাজ করে?

```yaml
# Pipeline এ এই secrets আছে:
env:
  ST_CICD_TOKEN:            ${{ secrets.ST_CICD_TOKEN }}
  ST_AWS_SECRET_ACCESS_KEY: ${{ secrets.ST_AWS_SECRET_ACCESS_KEY }}
  SLACK_WEBHOOK_URL:        ${{ secrets.SLACK_WEBHOOK }}
  CODECOMMITSSHKEY:         ${{ secrets.CODECOMMITSSHKEY }}
```

**ভেতরে যা হয়:**

```
১. GitHub Settings এ secret add করা হয়েছে
   → GitHub RSA key দিয়ে encrypt করে store করে

২. Job চালু হলে
   → Runner একটা temporary token পায়
   → সেই token দিয়ে GitHub API থেকে
     encrypted secret নামায়
   → Runner memory তে decrypt করে
   → Environment variable হিসেবে set করে

৩. Log এ কখনো দেখায় না
   → GitHub automatically *** দিয়ে replace করে
```

```bash
# এটা করলেও:
- run: echo ${{ secrets.ST_CICD_TOKEN }}

# Log এ দেখাবে:
# echo ***
```

> 💡 `vars.*` vs `secrets.*`:
> `ST_ECR_REPOSITORY`, `ST_AWS_REGION` → public, vars এ
> `ST_AWS_SECRET_ACCESS_KEY`, `SLACK_WEBHOOK` → sensitive, secrets এ

---

## Branch → Environment Mapping — কীভাবে কাজ করে?

এটা এই pipeline এর সবচেয়ে চালাক অংশ:

```bash
declare -A branches=( 
  ["refs/heads/develop"]="development"
  ["refs/heads/staging"]="staging"
  ["refs/heads/master"]="production"
)

for branch in "${!branches[@]}"; do
  if [ "${GITHUB_REF}" == "$branch" ]; then
    ENVIRONMENT=${branches[$branch]}
    break
  fi
done
```

**Technical flow:**

```
develop push হলো:
  GITHUB_REF = "refs/heads/develop"
  branches map এ match → ENVIRONMENT = "development"
  ↓
  ECR path: 123456.ecr.aws/development/flight-engine-b2c
  Dockerfile: Dockerfile.development
  GitOps repo: gitops-development.git

master push হলো:
  GITHUB_REF = "refs/heads/master"
  branches map এ match → ENVIRONMENT = "production"
  ↓
  ECR path: 123456.ecr.aws/production/flight-engine-b2c
  Dockerfile: Dockerfile.production
  GitOps repo: cicd.git
  + CodeCommit backup চালু হবে (only master)
```

---

## Docker Build — Layer Cache কীভাবে কাজ করে?

```bash
# Pipeline এ দুইটা project এর জন্য build হয়:
PROJECTS=("flight-engine-b2c" "flight-engine-b2b")

for PROJECT in "${PROJECTS[@]}"; do
  docker build -t $ECR_REPO:${{ github.sha }} \
               -f Dockerfile.${ENVIRONMENT} .
  docker push $ECR_REPO:${{ github.sha }}
done
```

**Technical flow (layer cache):**

```
Dockerfile.development এর layers:
  FROM node:18            → Layer 1 (300MB) — প্রায় কখনো বদলায় না
  COPY package*.json .    → Layer 2 (1KB)   — মাঝেমধ্যে বদলায়
  RUN npm install         → Layer 3 (150MB) — package.json বদলালে বদলায়
  COPY . .                → Layer 4 (10MB)  — প্রায়ই বদলায়
  CMD ["node", "server"]  → Layer 5 (1KB)

শুধু business logic বদলালে (Layer 4):
  Layer 1: cache hit ✅ skip (300MB download নেই)
  Layer 2: cache hit ✅ skip
  Layer 3: cache hit ✅ skip (npm install নেই!)
  Layer 4: cache miss ❌ rebuild (10MB)
  Layer 5: rebuild

সময়: ৩০ সেকেন্ড (cache ছাড়া ৫ মিনিট লাগতো)
```

---

## ECR Auto-Create — চালাক trick

```bash
# Repository না থাকলে নিজেই বানায়:
if ! aws ecr describe-repositories \
     --repository-names "${ENVIRONMENT}/${PROJECT}" \
     > /dev/null 2>&1; then
  aws ecr create-repository \
     --repository-name "${ENVIRONMENT}/${PROJECT}"
fi
```

**Technical flow:**

```
aws ecr describe-repositories → exists? 
  ✅ আছে → এগিয়ে যাও
  ❌ নেই → aws ecr create-repository
            নতুন repo বানাও
            তারপর এগিয়ে যাও

> /dev/null 2>&1 মানে:
  stdout → /dev/null (আবর্জনায় ফেলো, দেখাবে না)
  stderr → stdout এ (error ও দেখাবে না)
  শুধু exit code টা দরকার
```

---

## GitOps Pattern — Kubernetes কে কীভাবে জানায়?

এটা এই pipeline এর সবচেয়ে interesting অংশ:

```
সাধারণ deploy:
  CI/CD → SSH → Server → docker run  (direct)

GitOps pattern (এই pipeline):
  CI/CD → GitOps repo update → Kubernetes নিজেই দেখে → deploy
```

**Technical flow:**

```bash
# ১. GitOps repo clone করো
git clone "https://$ST_CICD_USER:$ST_CICD_TOKEN@github.com/
           Applicationnet/gitops-development.git"

# ২. Kustomize install করো
curl -s "https://raw.githubusercontent.com/
         kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash

# ৩. Image tag update করো
kustomize edit set image \
  flight-engine-b2c=123456.ecr.aws/development/flight-engine-b2c:a1b2c3

# ৪. Commit এবং push করো
git commit -am "[Github] updating development images to a1b2c3"
git push origin main
```

```
GitOps repo এ change গেলো
        ↓
ArgoCD/FluxCD (Kubernetes operator) দেখলো
"নতুন image tag এসেছে!"
        ↓
Kubernetes cluster এ নতুন image pull করলো
        ↓
Rolling update শুরু হলো:
  [old][old][old]
  [old][old][new]
  [old][new][new]
  [new][new][new]  ✅ zero downtime!
```

---

## CodeCommit Backup — শুধু Master এ

```yaml
- name: BACKUP CODE to AWS CODE-COMMIT
  uses: pixta-dev/repository-mirroring-action@v1
  if: github.ref == 'refs/heads/master'   # ← শুধু production deploy এ
  with:
    target_repo_url:
      ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/Application-flight-engine
    ssh_private_key: ${{ secrets.CODECOMMITSSHKEY }}
    ssh_username: ${{ secrets.CODECOMMITSSHKEYID }}
```

**Technical flow:**

```
Runner এ:
  private key → temp file এ লিখলো (/tmp/key)
  chmod 600 /tmp/key
  git mirror → AWS CodeCommit এ push
  temp key file মুছে ফেললো

কেন শুধু master?
  develop/staging → experiment, বারবার বদলায়
  master → stable production code → backup রাখার দরকার আছে

কেন CodeCommit?
  GitHub down হলেও AWS CodeCommit এ code থাকবে
  Disaster recovery! 🛡️
```

---

## Slack Notification — সবসময় জানাও

```yaml
- uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    fields: repo,message,commit,author,action,eventName,ref,workflow,job,took
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
  if: always()   # ← এটাই গুরুত্বপূর্ণ
```

**`if: always()` কেন দরকার?**

```
if: always() ছাড়া:
  step 4 এ docker build fail হলো ❌
  → বাকি সব steps skip
  → Slack notification ও skip
  → Team জানতেই পারলো না! 😱

if: always() দিলে:
  step 4 এ docker build fail হলো ❌
  → Slack step চলবেই
  → Team এ notification যাবে ❌ Build Failed!
  → কেউ fix করতে পারবে ✅
```

**Slack এ যা দেখাবে:**

```
✅ SUCCESS / ❌ FAILURE
Repo:      Applicationnet/flight-engine
Message:   fix: booking API timeout issue
Commit:    a1b2c3d
Author:    developer-name
Branch:    refs/heads/develop
Workflow:  Build & Push Docker Image
Took:      4m 32s
```

---

## পুরো Technical Flow — একনজরে

```
Developer: git push origin develop
          │
          ▼
GitHub Webhook fire:
{event: "push", ref: "refs/heads/develop", sha: "a1b2c3"}
          │
          ▼
Workflow trigger match: on.push.branches: [develop] ✅
          │
          ▼
Runner VM তৈরি: fresh Ubuntu Latest
          │
          ▼
Step 1: Checkout
  git clone flight-engine → /home/runner/work/
          │
          ▼
Step 2: AWS Credentials Configure
  access key + secret → AWS SDK configure
          │
          ▼
Step 3: ECR Login
  docker login → 123456.dkr.ecr.ap-southeast-1.amazonaws.com
          │
          ▼
Step 4: Docker Build & Push (loop: b2c + b2b)
  ENVIRONMENT = "development"  (develop branch বলে)
  ┌─────────────────────────────────────────┐
  │ ECR repo exist? → না → create করো      │
  │ docker build -f Dockerfile.development  │
  │   Layer 1: cache hit ✅                 │
  │   Layer 2: cache hit ✅                 │
  │   Layer 3: cache hit ✅                 │
  │   Layer 4: rebuild ❌ (code বদলেছে)    │
  │ docker push → ECR                       │
  │ (b2c done, এখন b2b এর জন্য repeat)    │
  └─────────────────────────────────────────┘
          │
          ▼
Step 5: GitOps Update (loop: b2c + b2b)
  git clone gitops-development.git
  kustomize edit set image → নতুন tag: a1b2c3
  git push → gitops-development/main
  ArgoCD দেখলো → K8s rolling update শুরু
          │
          ▼
Step 6: CodeCommit Backup
  if master? → না (develop branch) → SKIP ⏭️
          │
          ▼
Step 7: Slack Notification (always!)
  job.status = "success" → ✅ Slack message
          │
          ▼
Runner VM destroy হলো
          │
          ▼
Flight Engine নতুন version live! 🚀
মোট সময়: ~৮-১২ মিনিট
Developer এর কাজ: শুধু git push (৩০ সেকেন্ড)
```

---

## এই Pipeline এর Best Practices

| Practice | কোথায় দেখা যাচ্ছে |
|---|---|
| Secret আলাদা রাখা | `secrets.*` — token, key, webhook |
| Per-environment আলাদা config | branches declare -A map |
| Immutable image tag | `github.sha` — কখনো overwrite হয় না |
| Auto resource create | ECR describe → create |
| GitOps pattern | Kustomize → git push → K8s |
| Zero downtime deploy | K8s rolling update |
| Backup strategy | CodeCommit mirror (master only) |
| Always notify | `if: always()` Slack |
| Multi-project support | PROJECTS array loop |
| Environment-specific Dockerfile | `Dockerfile.development/production` |

> এটা একটা **production-grade** CI/CD pipeline — Application এর মতো real company তে ঠিক এভাবেই কাজ হয়! 🚀


---

## প্রথমে বুঝি — কী হয় যখন `git push` করো?

```
তোমার PC                GitHub Server           তোমার Server
─────────               ─────────────           ─────────────
git push ──────────────► code receive
                         event fire ──────────► GitHub Actions Runner চালু
                                                fresh Ubuntu VM তৈরি
                                                কোড clone
                                                test run
                                                docker build
                                                docker push
                                                SSH → deploy
                                                health check
```

GitHub internally একটা **webhook event** fire করে। সেই event GitHub Actions কে trigger করে। Actions একটা **fresh virtual machine** (runner) তৈরি করে এবং তোমার `.yml` ফাইল পড়ে কাজ শুরু করে।

---

## Runner কী জিনিস?

Runner হলো GitHub এর একটা **virtual machine** — যেখানে তোমার সব কাজ চলে।

```
GitHub Hosted Runner (default):
├── OS: Ubuntu 22.04 / Windows / macOS
├── RAM: 7 GB
├── CPU: 2 core
├── Disk: 14 GB
├── Pre-installed: Docker, Node, Python, Java, git...
└── প্রতিটা job এ FRESH — আগের কিছু থাকে না
```

প্রতিটা job শেষে VM destroy হয়ে যায়। পরের job এ একদম নতুন VM।

```yaml
runs-on: ubuntu-latest   # GitHub এর VM ব্যবহার করো (free)
runs-on: self-hosted     # তোমার নিজের server কে runner বানাও
```

---

## Webhook → Event → Trigger — ভেতরে কী হয়?

```
তুমি git push করলে
        ↓
GitHub এর git server কোড receive করলো
        ↓
GitHub internally event generate করলো:
{
  "event": "push",
  "ref": "refs/heads/main",
  "commits": [...],
  "pusher": { "name": "তুমি" },
  "repository": { "name": "myapp" }
}
        ↓
এই event .github/workflows/*.yml এর
"on:" section এর সাথে match করলো
        ↓
Match হলে → Runner queue তে job add হলো
        ↓
Available runner এ job চালু হলো
```

---

## একটা Step এ আসলে কী হয়?

```yaml
steps:
  - name: Install dependencies
    run: npm ci
```

এই একটা step এ যা হয়:

```
Runner এর shell (bash) এ:
  ┌─────────────────────────────────┐
  │  $ npm ci                       │
  │                                 │
  │  > added 847 packages           │
  │  > exit code: 0                 │
  └─────────────────────────────────┘

exit code 0   → success → পরের step চলবে
exit code 1+  → failure → job বন্ধ, error notify
```

প্রতিটা `run:` আসলে runner এর bash এ একটা script। exit code 0 হলে pass, অন্য কিছু হলে fail।

---

## actions/checkout — ভেতরে কী করে?

```yaml
- uses: actions/checkout@v3
```

এটা আসলে একটা Node.js script যা এই কাজ করে:

```bash
# internally যা হয়:
git clone https://github.com/তোমার/repo.git .
git checkout abc123def456  # যে commit push হয়েছে
```

Runner এ তোমার কোড নামিয়ে আনে। এটা ছাড়া runner এ কোড থাকে না।

---

## Context & Expression — Dynamic Value কীভাবে আসে?

```yaml
- run: echo "Branch: ${{ github.ref }}"
- run: echo "Commit: ${{ github.sha }}"
- run: echo "Actor: ${{ github.actor }}"
```

GitHub Actions এ `${{ }}` হলো **expression syntax**। এগুলো runtime এ replace হয়।

**গুরুত্বপূর্ণ contexts:**

```
github.ref        → refs/heads/main
github.sha        → a1b2c3d4e5f6...  (full commit hash)
github.actor      → তোমার GitHub username
github.event_name → push / pull_request / schedule
github.workspace  → /home/runner/work/myapp/myapp
                    (কোড কোথায় আছে runner এ)

secrets.MY_SECRET → GitHub Secrets থেকে encrypted value
env.MY_VAR        → Environment variable
```

---

## Secrets — Technically কীভাবে কাজ করে?

```yaml
- name: Login to Docker Hub
  with:
    password: ${{ secrets.DOCKER_PASSWORD }}
```

**ভেতরে যা হয়:**

```
১. তুমি GitHub এ secret add করলে
   → GitHub RSA key দিয়ে encrypt করে store করে

২. Job চালু হলে
   → Runner একটা temporary token পায়
   → সেই token দিয়ে GitHub API থেকে
     encrypted secret নামায়
   → Runner memory তে decrypt করে
   → Environment variable হিসেবে set করে

৩. Log এ কখনো দেখায় না
   → GitHub automatically *** দিয়ে replace করে
   → এমনকি তুমি `echo $SECRET` করলেও *** দেখাবে
```

```bash
# এটা করলেও:
- run: echo ${{ secrets.DOCKER_PASSWORD }}

# Log এ দেখাবে:
# echo ***
```

---

## Job এর মধ্যে Data Share — Outputs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.set_tag.outputs.tag }}

    steps:
      - name: Set image tag
        id: set_tag
        run: |
          TAG="myapp:${{ github.sha }}"
          echo "tag=$TAG" >> $GITHUB_OUTPUT
          # $GITHUB_OUTPUT একটা special file
          # এখানে লিখলে job output হয়ে যায়

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Use tag from build job
        run: |
          echo "Deploying: ${{ needs.build.outputs.image_tag }}"
          docker pull ${{ needs.build.outputs.image_tag }}
```

`$GITHUB_OUTPUT` হলো runner এ একটা special file। এখানে `key=value` লিখলে সেটা job output হয়ে যায় এবং অন্য job এ `needs.jobname.outputs.key` দিয়ে access করা যায়।

---

## Cache — কীভাবে Build দ্রুত হয়?

```yaml
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}
    restore-keys: npm-
```

**Technical flow:**

```
প্রথমবার চালালে:
  package-lock.json এর hash → abc123
  cache key → "npm-abc123"
  ~/.npm folder → GitHub cache server এ save

দ্বিতীয়বার চালালে:
  package-lock.json বদলায়নি → hash এখনো abc123
  cache key → "npm-abc123" → match!
  GitHub cache server থেকে ~/.npm restore
  npm ci → already cached → ৩০ সেকেন্ডে শেষ

package-lock.json বদলালে:
  নতুন hash → def456
  cache miss → আবার fresh install
  নতুন cache save → "npm-def456"
```

`hashFiles()` function package-lock.json এর MD5/SHA hash বের করে। ফাইল বদলালে hash বদলায়, cache miss হয়।

---

## Docker Layer Cache — Build দ্রুত করার আসল কারণ

```yaml
- name: Build and push
  uses: docker/build-push-action@v4
  with:
    cache-from: type=registry,ref=myapp:latest
    cache-to: type=inline
```

**Technical flow:**

```
Dockerfile:
  FROM node:18          → Layer 1 (300MB) — প্রায় কখনো বদলায় না
  COPY package*.json .  → Layer 2 (1KB)   — মাঝেমধ্যে বদলায়
  RUN npm install       → Layer 3 (150MB) — package.json বদলালে বদলায়
  COPY . .              → Layer 4 (10MB)  — প্রায়ই বদলায়
  CMD ["node", "app"]   → Layer 5 (1KB)   — মাঝেমধ্যে বদলায়

শুধু কোড বদলালে (Layer 4):
  Layer 1: cache hit ✅ skip (300MB download নেই)
  Layer 2: cache hit ✅ skip
  Layer 3: cache hit ✅ skip (npm install নেই!)
  Layer 4: cache miss ❌ rebuild (10MB)
  Layer 5: rebuild (কারণ 4 বদলেছে)

সময়: ৩০ সেকেন্ড (আগে ৫ মিনিট লাগতো)
```

---

## Services — Job এ Database কীভাবে চলে?

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: --health-cmd pg_isready
```

**Technical flow:**

```
Runner VM চালু হলো
       ↓
Docker daemon চালু হলো (runner এ pre-installed)
       ↓
postgres:15 image pull হলো
       ↓
postgres container চালু হলো
       ↓
health check: pg_isready
  → not ready → ১০ সেকেন্ড wait
  → not ready → আবার check
  → ready ✅
       ↓
তোমার steps চালু হলো
DATABASE_URL=postgres://localhost:5432/...
       ↓
job শেষে postgres container বন্ধ হলো
```

Service container ও runner এর steps একই Docker network এ থাকে। তাই `localhost:5432` দিয়ে access করা যায়।

---

## SSH Deploy — Technically কী হয়?

```yaml
- uses: appleboy/ssh-action@v0.1.10
  with:
    host: ${{ secrets.SERVER_IP }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    script: |
      docker pull myapp:latest
      docker stop myapp || true
      docker run -d --name myapp myapp:latest
```

**Technical flow:**

```
Runner এ:
  private key → temp file এ লিখলো (/tmp/key)
  chmod 600 /tmp/key
  ssh -i /tmp/key ubuntu@SERVER_IP \
    "docker pull myapp:latest && ..."
       ↓
Server এ:
  command execute হলো
  output → runner এ ফিরে এলো
  exit code → success/failure
       ↓
Runner এ:
  temp key file মুছে ফেললো
  exit code check করলো
```

`|| true` কেন? `docker stop myapp` fail করে যদি container না থাকে। `|| true` দিলে fail হলেও script বন্ধ হয় না।

---

## Health Check — Deploy সফল কিনা বুঝবো কীভাবে?

```yaml
- name: Health check
  run: |
    echo "Waiting for app to start..."
    sleep 15

    for i in {1..5}; do
      STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
               http://${{ secrets.SERVER_IP }}:5000/health)

      if [ "$STATUS" = "200" ]; then
        echo "✅ App is healthy!"
        exit 0
      fi

      echo "Attempt $i failed (status: $STATUS), retrying..."
      sleep 10
    done

    echo "❌ Health check failed!"
    exit 1
```

**Technical flow:**

```
curl request → /health endpoint
                   ↓
HTTP 200 → অ্যাপ ঠিকমতো চলছে → exit 0 → job success
HTTP 500 → অ্যাপে সমস্যা
HTTP 000 → অ্যাপ চালুই হয়নি
                   ↓
5 বার try করবে, তারপরও fail → exit 1 → job fail
                   ↓
GitHub notify করবে, আগের version rollback করো
```

---

## Rollback — Automatically আগের Version এ ফেরা

```yaml
deploy:
  steps:
    - name: Save current version
      run: |
        CURRENT=$(docker ps --filter name=myapp \
                  --format "{{.Image}}")
        echo "PREVIOUS_IMAGE=$CURRENT" >> $GITHUB_ENV

    - name: Deploy new version
      run: |
        docker pull myapp:${{ github.sha }}
        docker stop myapp || true
        docker run -d --name myapp \
          myapp:${{ github.sha }}

    - name: Health check
      id: health
      run: curl --fail http://server/health

    - name: Rollback if failed
      if: failure() && steps.health.conclusion == 'failure'
      run: |
        echo "❌ Deploy failed! Rolling back..."
        docker stop myapp || true
        docker run -d --name myapp ${{ env.PREVIOUS_IMAGE }}
        echo "✅ Rolled back to ${{ env.PREVIOUS_IMAGE }}"
```

---

## পুরো Technical Flow — একনজরে

```
তুমি: git push origin main
          │
          ▼
GitHub git server: কোড receive
          │
          ▼
Webhook event fire:
{event: "push", ref: "main", sha: "abc123"}
          │
          ▼
Workflow trigger match: on.push.branches: [main]
          │
          ▼
Runner VM তৈরি: fresh Ubuntu 22.04
          │
     ┌────┴────┐
     ▼         ▼
  JOB: test  (parallel হতে পারে)
     │
     ├─ Step: checkout → git clone
     ├─ Step: setup-node → node 18 install
     ├─ Step: cache → npm cache restore
     ├─ Step: npm ci → dependencies install
     ├─ Step: service postgres → container চালু
     ├─ Step: npm test → test run
     │         │
     │    exit 0 ✅
     │
     ▼
  JOB: build (needs: test)
  নতুন fresh VM
     │
     ├─ Step: checkout → git clone
     ├─ Step: docker login → Docker Hub auth
     ├─ Step: docker build → layer cache check
     │         ├─ Layer 1: cache hit ✅
     │         ├─ Layer 2: cache hit ✅
     │         ├─ Layer 3: cache hit ✅
     │         └─ Layer 4: rebuild ❌ (কোড বদলেছে)
     ├─ Step: docker push → Docker Hub এ upload
     │
     ▼
  JOB: deploy (needs: build)
  নতুন fresh VM
     │
     ├─ Step: SSH connect → private key দিয়ে
     ├─ Step: docker pull → নতুন image নামাও
     ├─ Step: docker stop → পুরনো container বন্ধ
     ├─ Step: docker run → নতুন container চালু
     ├─ Step: health check → HTTP 200? ✅
     │
     ▼
  সব VM destroy হলো
  GitHub Actions history তে record থাকলো
          │
          ▼
তোমার অ্যাপ live! 🚀
মোট সময়: ~১৫ মিনিট
তোমার কাজ: শুধু git push (৩০ সেকেন্ড)
```

---

সংক্ষেপে — `git push` করলে GitHub **webhook** fire হয়, **fresh VM** তৈরি হয়, তোমার `.yml` পড়ে **step by step** কাজ করে, **exit code** দিয়ে success/failure বোঝে, **secrets** encrypted রাখে, **cache** দিয়ে দ্রুত করে, আর শেষে **health check** দিয়ে নিশ্চিত করে সব ঠিক আছে।

AWS বা Kubernetes — কোনটা দেখতে চাও?
