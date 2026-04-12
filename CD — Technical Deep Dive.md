# CI/CD — Technical Deep Dive

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
