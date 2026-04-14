# GitHub Actions — Complete Guide 🚀
---

# 🟢 FOUNDATION

---

## ১. Workflow Structure — মূল কাঠামো

### কেন দরকার?
GitHub Actions কে বলতে হয় — **কখন**, **কোথায়**, **কী করবে**।
Workflow file হলো সেই instruction sheet।

### কোথায় রাখবে?
```
তোমার project/
└── .github/
    └── workflows/
        ├── build.yml      ← এখানে রাখো
        ├── deploy.yml
        └── test.yml
```

### পুরো structure:

```yaml
# ─────────────────────────────────────────────
# name: Workflow এর নাম (GitHub UI তে দেখাবে)
# ─────────────────────────────────────────────
name: Build and Deploy

# ─────────────────────────────────────────────
# on: কখন এই workflow চালু হবে
# ─────────────────────────────────────────────
on:
  push:
    branches: [main]

# ─────────────────────────────────────────────
# jobs: কী কী কাজ করবে
# একটা workflow এ একাধিক job থাকতে পারে
# ─────────────────────────────────────────────
jobs:

  # job এর নাম (তুমি দেবে)
  build:

    # কোন machine এ চলবে
    runs-on: ubuntu-latest

    # এই job এ কী কী step আছে
    steps:

      # Step 1: Code নামাও
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Node.js setup করো
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      # Step 3: Dependencies install করো
      - name: Install dependencies
        run: npm install

      # Step 4: Test চালাও
      - name: Run tests
        run: npm test

      # Step 5: Build করো
      - name: Build project
        run: npm run build
```

### Key concepts:
```
Workflow  → পুরো .yml file টা
Job       → workflow এর ভেতরে একটা কাজের unit
Step      → job এর ভেতরে একটা ছোট কাজ
Runner    → যে machine এ job চলে
Action    → ready-made step (uses: দিয়ে)
Command   → shell command (run: দিয়ে)
```

---

## ২. Triggers (on:) — কখন চালু হবে?

### কেন দরকার?
সব সময় workflow চালাতে চাই না।
নির্দিষ্ট event এ চালু হোক।

### সব গুরুত্বপূর্ণ triggers:

```yaml
on:

  # ── Push হলে ──
  push:
    branches:
      - main           # শুধু main branch এ push হলে
      - develop        # অথবা develop branch এ
    branches-ignore:
      - 'temp-*'       # temp- দিয়ে শুরু branch এ push হলে না
    paths:
      - 'src/**'       # শুধু src/ folder এর file বদলালে
      - '**.js'        # যেকোনো .js file বদলালে
    paths-ignore:
      - '**.md'        # .md file বদলালে চালাবে না

  # ── Pull Request হলে ──
  pull_request:
    branches: [main]   # main এ PR হলে
    types:
      - opened         # নতুন PR খুললে
      - synchronize    # PR এ নতুন commit আসলে
      - reopened       # বন্ধ PR আবার খুললে

  # ── Manually চালাতে চাইলে ──
  workflow_dispatch:   # GitHub UI তে "Run workflow" button দেখাবে

  # ── নির্দিষ্ট সময়ে ──
  schedule:
    - cron: '0 2 * * *'  # প্রতিদিন রাত ২টায় (UTC)

  # ── Release হলে ──
  release:
    types: [published]   # Release publish হলে

  # ── অন্য workflow call করলে ──
  workflow_call:         # Reusable workflow এর জন্য

  # ── একাধিক trigger একসাথে ──
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
```

### Real example — কোন project এ কোনটা use করবে:

```yaml
# ছোট personal project:
on: [push]   # যেকোনো push এ চালাও

# Team project:
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# Production deployment:
on:
  push:
    branches: [main]        # main এ push হলে auto deploy
  workflow_dispatch:        # দরকারে manually deploy
  release:
    types: [published]      # release দিলে deploy
```

---

## ৩. Jobs & Steps — কাজের ভাগ

### কেন দরকার?
বড় কাজকে ছোট ছোট ভাগে ভাগ করতে।
কিছু কাজ parallel এ, কিছু sequence এ করতে।

### Jobs বনাম Steps:

```
Job A ──────────────────────────────────
  Step 1 → Step 2 → Step 3 → Step 4
  (একটার পর একটা, same machine)

Job B ──────────────────────────────────  ← Job A এর সাথে PARALLEL চলে
  Step 1 → Step 2
  (আলাদা fresh machine)
```

### Step এর দুই ধরন:

```yaml
steps:
  # ── ধরন ১: uses (ready-made action) ──
  - name: Checkout code
    uses: actions/checkout@v4    # GitHub Marketplace থেকে
    with:                        # action এর parameters
      fetch-depth: 0             # সব history নামাও

  # ── ধরন ২: run (shell command) ──
  - name: Install and test
    run: |                       # | মানে multiline command
      npm install
      npm test
      npm run build

  # ── Single line command ──
  - name: Print hello
    run: echo "Hello World"

  # ── Step এর extra options ──
  - name: Risky step
    run: might-fail.sh
    continue-on-error: true      # fail হলেও পরের step চলবে
    timeout-minutes: 10          # ১০ মিনিটের বেশি লাগলে cancel
    working-directory: ./backend # এই folder এ command চালাও
    shell: bash                  # কোন shell use করবে
```

### Multiple Jobs:

```yaml
jobs:
  # Job 1: Test করো
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  # Job 2: Lint করো (Job 1 এর সাথে PARALLEL)
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  # Job 3: Build করো (test শেষ হলে)
  build:
    needs: [test, lint]          # test আর lint দুটো শেষ হলে তবেই
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

---

## ৪. Runners — কোথায় চলে?

### কেন দরকার?
তোমার code কোনো একটা machine এ run করতে হবে।
Runner হলো সেই machine।

### GitHub-hosted Runners (default):

```yaml
jobs:
  build:
    # ── Linux (সবচেয়ে popular, সস্তা) ──
    runs-on: ubuntu-latest      # Ubuntu 24.04
    runs-on: ubuntu-22.04       # নির্দিষ্ট version

    # ── Windows ──
    runs-on: windows-latest     # Windows Server 2022

    # ── macOS (iOS app build এ লাগে) ──
    runs-on: macos-latest       # macOS 14

    # ── Larger runners (paid) ──
    runs-on: ubuntu-latest-4-cores   # 4 core, 16GB RAM
    runs-on: ubuntu-latest-8-cores   # 8 core, 32GB RAM
```

### GitHub-hosted runner এ কী কী আছে?

```
Pre-installed tools:
├── Docker
├── Git
├── Node.js (multiple versions)
├── Python (multiple versions)
├── Java
├── AWS CLI
├── kubectl
└── আরো অনেক কিছু

Hardware:
├── CPU: 2 core (standard)
├── RAM: 7 GB
├── Disk: 14 GB
└── প্রতিটা job এ FRESH machine
```

### Self-hosted Runner (নিজের server):

```yaml
jobs:
  build:
    runs-on: self-hosted          # নিজের server এ চালাও
    # অথবা label দিয়ে:
    runs-on: [self-hosted, linux, gpu]  # GPU server এ চালাও
```

```bash
# নিজের server এ setup করতে:
# GitHub → Settings → Actions → Runners → New self-hosted runner

# ১. Runner software নামাও
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/latest/download/...
tar xzf actions-runner-linux-x64.tar.gz

# ২. Register করো
./config.sh --url https://github.com/YOUR_ORG/YOUR_REPO \
            --token YOUR_TOKEN

# ৩. Service হিসেবে চালাও (সবসময় চলতে থাকবে)
sudo ./svc.sh install
sudo ./svc.sh start
```

### কোনটা কখন use করবে?

```
GitHub-hosted → বেশিরভাগ project এ
Self-hosted   → Private network access দরকার হলে
              → Special hardware লাগলে (GPU)
              → Cost বাঁচাতে চাইলে
              → GitHub runner যথেষ্ট powerful না হলে
```

---

## ৫. Secrets & Variables — Password লুকিয়ে রাখো

### কেন দরকার?
AWS key, database password, API token —
এগুলো code এ লেখা যাবে না।
কেউ দেখতে পাবে না এমনভাবে রাখতে হবে।

### Secrets vs Variables:

```
secrets.*  → sensitive data (password, key, token)
           → encrypted, কেউ দেখতে পাবে না
           → log এ *** দেখাবে

vars.*     → non-sensitive config (region, repo URL)
           → plain text, দেখা যায়
           → log এ actual value দেখাবে
```

### কীভাবে set করবে?

```
GitHub → Repository → Settings → Secrets and variables → Actions

Secrets:
  DB_PASSWORD = "super_secret_123"
  AWS_SECRET_KEY = "wJalrXUtnFEMI..."
  SLACK_WEBHOOK = "https://hooks.slack.com/..."

Variables:
  AWS_REGION = "ap-southeast-1"
  APP_NAME = "flight-engine"
  ECR_REPOSITORY = "123456.dkr.ecr.aws"
```

### Workflow এ use করা:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest

    # ── Job level env (এই job এর সব step এ পাবে) ──
    env:
      APP_NAME: ${{ vars.APP_NAME }}                    # Variable
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}           # Secret
      AWS_REGION: ${{ vars.AWS_REGION }}                # Variable
      AWS_SECRET: ${{ secrets.AWS_SECRET_ACCESS_KEY }}  # Secret

    steps:
      # ── Step level env (শুধু এই step এ) ──
      - name: Deploy
        env:
          DEPLOY_ENV: production
        run: |
          echo "App: $APP_NAME"          # Variable → দেখা যাবে
          echo "Region: $AWS_REGION"     # Variable → দেখা যাবে
          echo "Pass: $DB_PASSWORD"      # Secret → *** দেখাবে
          ./deploy.sh

      # ── with এ secret দেওয়া ──
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}   # encrypted হয়ে যাবে
```

### Secret কীভাবে কাজ করে (ভেতরে):

```
তুমি GitHub এ secret add করলে:
  → GitHub RSA key দিয়ে encrypt করে store করে

Job চালু হলে:
  → Runner temporary token পায়
  → GitHub API থেকে encrypted secret নামায়
  → Runner memory তে decrypt করে
  → Environment variable হিসেবে set করে

Log এ:
  run: echo $DB_PASSWORD
  # Output: ***   ← কখনো দেখাবে না
```

### Secret এর levels:

```
Repository secret  → শুধু এই repo তে
Environment secret → নির্দিষ্ট environment এ (production/staging)
Organization secret → পুরো organization এর সব repo তে
```

---

## ৬. Expressions & Contexts — Dynamic Values

### কেন দরকার?
Workflow এ dynamic value দরকার।
কোন branch, কে push করলো, commit hash কী —
এগুলো runtime এ জানতে হয়।

### Expression syntax:

```yaml
${{ expression }}   # এই ভেতরে যা লিখবে evaluate হবে
```

### সব গুরুত্বপূর্ণ contexts:

```yaml
steps:
  - run: |
      # ── github context (repository info) ──
      echo "Repo: ${{ github.repository }}"        # org/repo-name
      echo "Branch: ${{ github.ref }}"             # refs/heads/main
      echo "Branch short: ${{ github.ref_name }}"  # main
      echo "Commit: ${{ github.sha }}"             # a1b2c3d4e5f6...
      echo "Short SHA: ${{ github.sha }}"          # a1b2c3d (first 7)
      echo "Actor: ${{ github.actor }}"            # যে push করেছে
      echo "Event: ${{ github.event_name }}"       # push/pull_request
      echo "Workspace: ${{ github.workspace }}"    # /home/runner/work/...
      echo "Run ID: ${{ github.run_id }}"          # unique run number

      # ── secrets context ──
      echo "${{ secrets.MY_SECRET }}"   # → ***

      # ── vars context ──
      echo "${{ vars.MY_VAR }}"         # → actual value

      # ── env context ──
      echo "${{ env.MY_ENV_VAR }}"      # → value set in env:

      # ── job context ──
      echo "${{ job.status }}"          # success/failure/cancelled

      # ── runner context ──
      echo "${{ runner.os }}"           # Linux/Windows/macOS
      echo "${{ runner.arch }}"         # X64/ARM64

      # ── steps context (আগের step এর output) ──
      echo "${{ steps.my-step-id.outputs.my-output }}"
```

### Functions — এগুলো expression এর ভেতরে use করা যায়:

```yaml
steps:
  - run: echo "test"
    # contains() — string এ কিছু আছে কিনা
    if: contains(github.ref, 'main')

  - run: echo "test"
    # startsWith() — কী দিয়ে শুরু
    if: startsWith(github.ref, 'refs/heads/feature')

  - run: echo "test"
    # endsWith() — কী দিয়ে শেষ
    if: endsWith(github.repository, 'flight-engine')

  - run: echo "test"
    # toJSON() — object কে JSON string করো
    run: echo '${{ toJSON(github) }}'

  - name: Hash file
    run: echo "hash"
    # hashFiles() — file এর hash বের করো (cache key তে use হয়)
    # ${{ hashFiles('package-lock.json') }} → abc123def456
```

---

## ৭. Conditional (if:) — শর্ত দিয়ে control করো

### কেন দরকার?
সব step সবসময় চালাতে চাই না।
- শুধু main branch এ deploy করবো
- শুধু fail হলে alert পাঠাবো
- শুধু PR এ test করবো

### Basic if:

```yaml
steps:
  # ── Branch check ──
  - name: Deploy to Production
    run: ./deploy.sh production
    if: github.ref == 'refs/heads/main'   # শুধু main branch এ

  - name: Deploy to Staging
    run: ./deploy.sh staging
    if: github.ref == 'refs/heads/develop'  # শুধু develop branch এ

  # ── Event check ──
  - name: Comment on PR
    run: echo "PR comment"
    if: github.event_name == 'pull_request'

  # ── Job status check ──
  - name: Notify on failure
    run: curl -X POST $SLACK_WEBHOOK -d '{"text":"❌ Build Failed!"}'
    if: failure()          # আগের কোনো step fail হলে

  - name: Always cleanup
    run: rm -rf /tmp/build
    if: always()           # সবসময় (success বা failure)

  - name: Only on success
    run: notify-success.sh
    if: success()          # সব step success হলে (default behavior)

  - name: Success or failure (but not cancelled)
    run: upload-report.sh
    if: success() || failure()

  # ── Actor check ──
  - name: Admin only step
    run: admin-task.sh
    if: github.actor == 'admin-username'

  # ── Complex conditions ──
  - name: Deploy only on main push (not PR)
    run: deploy.sh
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

  - name: Skip for draft PR
    run: full-test.sh
    if: github.event.pull_request.draft == false
```

### Job level if:

```yaml
jobs:
  # ── শুধু main branch এ এই job চলবে ──
  deploy:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - run: deploy.sh

  # ── PR হলে এই job skip হবে ──
  release:
    runs-on: ubuntu-latest
    if: github.event_name != 'pull_request'
    steps:
      - run: release.sh
```

---

# 🔵 INTERMEDIATE

---

## ৮. Job Dependencies (needs:) — Job এর order

### কেন দরকার?
Test পাস না হলে deploy করা ঠিক না।
Build না হলে publish করা যাবে না।
`needs:` দিয়ে বলো কোন job আগে শেষ হতে হবে।

### Basic needs:

```yaml
jobs:
  # Step 1: Test (আগে চলবে)
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  # Step 2: Lint (test এর সাথে PARALLEL চলবে)
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  # Step 3: Build (test আর lint দুটো শেষ হলে)
  build:
    needs: [test, lint]      # দুটোই pass হতে হবে
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build

  # Step 4: Deploy (build শেষ হলে)
  deploy:
    needs: build             # একটাই লাগলে array লাগে না
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### চলার order:

```
test ──────┐
           ├──→ build ──→ deploy
lint ──────┘

test আর lint parallel এ চলে।
দুটো শেষ হলে build চলে।
build শেষ হলে deploy চলে।
```

### Failed job এর পরেও চালাতে চাইলে:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test   # ধরো এটা fail করলো

  # test fail হলেও এই job চালাও
  notify:
    needs: test
    runs-on: ubuntu-latest
    if: always()          # ← এটা দিলে test fail হলেও চলবে
    steps:
      - run: send-notification.sh
```

---

## ৯. Job Outputs — Jobs এর মধ্যে data share

### কেন দরকার?
Job A তে version বের করলে, Job B তে সেই version দরকার।
কিন্তু প্রতিটা job আলাদা fresh machine —
সরাসরি data share হয় না।
Outputs দিয়ে pass করতে হয়।

```yaml
jobs:
  # ── Job 1: Data বের করো ──
  prepare:
    runs-on: ubuntu-latest

    # এই job কী output দেবে তা declare করো
    outputs:
      version: ${{ steps.get-ver.outputs.version }}
      image_tag: ${{ steps.get-ver.outputs.tag }}
      environment: ${{ steps.get-env.outputs.env }}

    steps:
      - uses: actions/checkout@v4

      - name: Get version and tag
        id: get-ver          # ← id দিতে হবে, outputs এ reference করতে
        run: |
          VERSION=$(cat package.json | python3 -c \
            "import sys,json; print(json.load(sys.stdin)['version'])")

          # $GITHUB_OUTPUT এ লিখলে output হয়ে যায়
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "tag=$VERSION-${{ github.sha }}" >> $GITHUB_OUTPUT

      - name: Detect environment
        id: get-env
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            echo "env=production" >> $GITHUB_OUTPUT
          else
            echo "env=staging" >> $GITHUB_OUTPUT
          fi

  # ── Job 2: Output use করো ──
  build:
    needs: prepare          # prepare আগে চলতে হবে
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build with version
        run: |
          # needs.JOB_NAME.outputs.OUTPUT_NAME দিয়ে access
          echo "Version: ${{ needs.prepare.outputs.version }}"
          echo "Tag: ${{ needs.prepare.outputs.image_tag }}"

          docker build \
            -t myapp:${{ needs.prepare.outputs.image_tag }} \
            --build-arg VERSION=${{ needs.prepare.outputs.version }} \
            .

  # ── Job 3: একই output use করো ──
  deploy:
    needs: [prepare, build]
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: |
          ENV="${{ needs.prepare.outputs.environment }}"
          TAG="${{ needs.prepare.outputs.image_tag }}"

          echo "Deploying $TAG to $ENV"
          kubectl set image deployment/myapp myapp=myapp:$TAG -n $ENV
```

---

## ১০. Artifacts — File share করো jobs এর মধ্যে

### কেন দরকার?
Job 1 এ build করলো, dist/ folder তৈরি হলো।
Job 2 তে deploy করতে হবে।
কিন্তু Job 2 fresh machine — dist/ নেই!

Artifact = Job 1 এর file → GitHub server এ upload → Job 2 download করে।

```yaml
jobs:
  # ── Job 1: Build করো ──
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build    # dist/ folder তৈরি হলো

      # dist/ folder upload করো
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output       # artifact এর নাম
          path: dist/              # কোন folder upload করবো
          retention-days: 7        # কতদিন রাখবো (default: 90)

      # একাধিক file/folder:
      - name: Upload multiple artifacts
        uses: actions/upload-artifact@v4
        with:
          name: all-outputs
          path: |
            dist/
            coverage/
            *.log

  # ── Job 2: Deploy করো (fresh machine!) ──
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      # dist/ নামিয়ে আনো
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: build-output     # build job এ যে নাম দিয়েছিলে
          path: dist/            # কোথায় রাখবো

      - name: Deploy
        run: |
          ls dist/    # এখন আছে! ✅
          ./deploy.sh dist/

  # ── Test Report আলাদা artifact হিসেবে রাখো ──
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test -- --coverage

      - name: Upload test coverage
        uses: actions/upload-artifact@v4
        if: always()    # fail হলেও upload করো
        with:
          name: coverage-report
          path: coverage/
```

### Artifact কোথায় দেখবে?

```
GitHub → Actions → কোনো workflow run → Artifacts section এ
(download করতে পারবে)
```

---

## ১১. Cache — Build দ্রুত করো

### কেন দরকার?
প্রতিটা job fresh machine।
তাই প্রতিবার `npm install` = ১০০০+ packages download।
সময় লাগে ২-৩ মিনিট।

Cache দিলে → packages save থাকবে → পরের বার ১০ সেকেন্ড!

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # ── npm cache ──
      - name: Cache npm packages
        uses: actions/cache@v4
        with:
          path: ~/.npm              # কোন folder cache করবো
          key: npm-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
          # key মানে: "Linux এ, package-lock.json এর hash abc123"
          # package-lock.json না বদলালে hash same → cache hit!
          restore-keys: |
            npm-${{ runner.os }}-   # partial match fallback
            npm-

      - run: npm ci    # cache hit হলে download হবে না!

      # ── Python pip cache ──
      - name: Cache pip packages
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: pip-${{ runner.os }}-${{ hashFiles('requirements.txt') }}

      - run: pip install -r requirements.txt

      # ── Maven cache (Java) ──
      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2
          key: maven-${{ runner.os }}-${{ hashFiles('pom.xml') }}

      - run: mvn install
```

### Cache কীভাবে কাজ করে:

```
প্রথমবার:
  package-lock.json hash → abc123
  cache key → "npm-Linux-abc123"
  ~/.npm folder → GitHub cache server এ save (500MB পর্যন্ত)
  npm install → সব download হলো

দ্বিতীয়বার (package-lock.json বদলায়নি):
  hash → এখনো abc123
  cache hit! → ~/.npm restore হলো
  npm install → already cached → ১০ সেকেন্ড!

package-lock.json বদলালে:
  নতুন hash → def456
  cache miss → fresh install
  নতুন cache save
```

### Setup-node এ built-in cache:

```yaml
# আলাদা cache action লাগে না!
- uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'          # automatically npm cache করবে
    # cache: 'yarn'       # yarn হলে
    # cache: 'pnpm'       # pnpm হলে
```

---

## ১২. Matrix Strategy — Parallel testing

### কেন দরকার?
তোমার app Node 16, 18, 20 তে কাজ করে কিনা test করতে হবে।
আলাদা আলাদা job লিখলে repetition হয়।
Matrix দিলে automatically সব combination এ test হবে।

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      # ── Basic matrix ──
      matrix:
        node-version: [16, 18, 20]    # এই তিনটা value তে চলবে
        # মোট ৩টা job তৈরি হবে

      # fail-fast: একটা fail হলে বাকিগুলো cancel করবে কিনা
      fail-fast: false     # false = বাকিগুলো চলতে দাও (default: true)

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}   # matrix value use করো

      - run: npm test

  # ── Multiple dimensions ──
  cross-platform-test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20]
        # মোট ৬টা job (3 OS × 2 Node versions)

    runs-on: ${{ matrix.os }}    # matrix থেকে os নাও
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test

  # ── কিছু combination বাদ দাও ──
  selective-test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [16, 18, 20]
        exclude:
          # এই combination বাদ (windows এ node 16 দরকার নেই)
          - os: windows-latest
            node: 16
        include:
          # এই extra combination যোগ করো
          - os: ubuntu-latest
            node: 20
            experimental: true    # extra variable যোগ করা যায়

    runs-on: ${{ matrix.os }}
    continue-on-error: ${{ matrix.experimental == true }}
    steps:
      - run: npm test
```

### চলার pattern:

```
node: [16, 18, 20] matrix দিলে:

Job 1: node-version=16  ─┐
Job 2: node-version=18  ─├─ সব parallel এ চলে
Job 3: node-version=20  ─┘

os + node matrix দিলে:
Job 1: ubuntu  + node18 ─┐
Job 2: ubuntu  + node20  │
Job 3: windows + node18  ├─ সব parallel এ চলে
Job 4: windows + node20  │
Job 5: macos   + node18  │
Job 6: macos   + node20 ─┘
```

---

## ১৩. Environments & Approval Gate

### কেন দরকার?
Production এ যাওয়ার আগে কেউ manually approve করুক।
Accidentally production deploy হওয়া ঠেকাতে।
Environment-specific secrets রাখতে।

### Setup (GitHub UI তে):

```
Repository → Settings → Environments → New environment

environment: production
  ✅ Required reviewers: [senior-dev, team-lead]
  ✅ Wait timer: 5 minutes
  ✅ Deployment branches: main only
  
  Secrets (production specific):
    DB_PASSWORD = "prod_password"   # staging এর আলাদা
    API_KEY = "prod_api_key"
```

### Workflow এ use করা:

```yaml
jobs:
  # ── Staging (auto deploy, no approval) ──
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging       # staging environment use করো
    steps:
      - name: Deploy to staging
        run: |
          echo "DB: ${{ secrets.DB_PASSWORD }}"   # staging DB password
          ./deploy.sh staging

  # ── Production (approval দরকার!) ──
  deploy-production:
    needs: deploy-staging      # staging আগে
    runs-on: ubuntu-latest
    environment: production    # ← approval gate এখানে!
    # কেউ approve না করলে এই job চলবে না
    # GitHub notification পাঠাবে reviewers দের
    steps:
      - name: Deploy to production
        run: |
          echo "DB: ${{ secrets.DB_PASSWORD }}"   # production DB password
          ./deploy.sh production
```

### Flow:

```
staging deploy হলো ✅
        ↓
GitHub notification পাঠালো:
"deploy-production job approval চাই!"
        ↓
Senior dev GitHub এ গেলো → Review deployments → Approve
        ↓
production deploy শুরু হলো 🚀
```

---

## ১৪. Concurrency Control — একসাথে দুটো না

### কেন দরকার?
দুইজন একই branch এ একসাথে push করলো।
দুটো deploy একসাথে চলছে।
একটা নতুন deploy শুরু হলে পুরনোটা cancel হোক।

```yaml
# ── Workflow level concurrency ──
concurrency:
  group: deploy-${{ github.ref }}
  # "deploy-refs/heads/main" — same group এ একটাই চলবে
  cancel-in-progress: true
  # নতুনটা আসলে পুরনোটা cancel করো

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh

# ── Job level concurrency ──
jobs:
  deploy:
    runs-on: ubuntu-latest
    concurrency:
      group: production-deploy    # এই group এ একটাই চলবে
      cancel-in-progress: false   # পুরনোটা শেষ হওয়ার জন্য wait করো
    steps:
      - run: ./deploy.sh
```

### বাস্তব scenario:

```
cancel-in-progress: true দিলে:
  Push 1 → deploy চলছে...
  Push 2 → Push 1 CANCEL হলো ❌
           Push 2 শুরু হলো ✅
  Result: Latest code deploy হবে (সবচেয়ে ভালো)

cancel-in-progress: false দিলে:
  Push 1 → deploy চলছে...
  Push 2 → Push 1 শেষ হওয়ার জন্য wait করছে
           Push 1 শেষ → Push 2 শুরু
  Result: সব deploy হবে, কোনো cancel নেই

কোনটা use করবে:
  Deploy job → cancel-in-progress: true (latest code চাই)
  Release job → cancel-in-progress: false (সব release হোক)
```

---

## ১৫. Manual Trigger (workflow_dispatch)

### কেন দরকার?
- Emergency deploy করতে হবে
- Specific version deploy করতে হবে
- Production deploy এ extra safety চাই
- Test manually চালাতে চাই

```yaml
on:
  push:
    branches: [main]     # auto trigger
  workflow_dispatch:      # manual trigger (button)
    inputs:
      environment:
        description: 'কোথায় deploy করবো?'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
        default: development

      image_tag:
        description: 'কোন image tag? (blank = latest commit)'
        required: false
        type: string

      skip_tests:
        description: 'Tests skip করবো? (emergency deploy)'
        required: false
        type: boolean
        default: false

      reason:
        description: 'Deploy এর কারণ (audit log)'
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Show deploy info
        run: |
          echo "Environment: ${{ inputs.environment }}"
          echo "Tag: ${{ inputs.image_tag || github.sha }}"
          echo "Skip tests: ${{ inputs.skip_tests }}"
          echo "Reason: ${{ inputs.reason }}"

      - name: Run tests (skip করা যাবে)
        if: inputs.skip_tests == false || github.event_name == 'push'
        run: npm test

      - name: Deploy
        run: |
          TAG="${{ inputs.image_tag || github.sha }}"
          ENV="${{ inputs.environment || 'development' }}"
          ./deploy.sh $ENV $TAG
```

### GitHub UI তে কীভাবে দেখাবে:

```
Actions → [Workflow name] → Run workflow ▼
┌────────────────────────────────────────┐
│ Use workflow from: Branch: main        │
│                                        │
│ কোথায় deploy করবো?                    │
│ [production ▼]                         │
│                                        │
│ কোন image tag?                         │
│ [v1.2.3________________]              │
│                                        │
│ Tests skip করবো?                      │
│ [ ] false                              │
│                                        │
│ Deploy এর কারণ:                       │
│ [Hotfix for payment bug_____]         │
│                                        │
│         [Run workflow]                 │
└────────────────────────────────────────┘
```

---

# 🟡 ADVANCED

---

## ১৬. Reusable Workflows — Template বানাও

### কেন দরকার?
১০টা repo তে একই deploy workflow।
একটায় bug fix করলে বাকি ৯টায় manually করতে হয়।

Reusable workflow = একবার লেখো, সবাই call করুক।

```yaml
# ─────────────────────────────────────────────────────
# File: .github/workflows/deploy-template.yml
# এটা template — directly চলে না, call করতে হয়
# ─────────────────────────────────────────────────────
on:
  workflow_call:              # ← এটাই বলে "আমি reusable"
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: string          # string, boolean, number
      image_tag:
        required: true
        type: string
      dry_run:
        required: false
        type: boolean
        default: false
    secrets:
      AWS_KEY:                # caller এর secret forward করো
        required: true
      AWS_SECRET:
        required: true
    outputs:                  # caller কে output দাও
      deploy_url:
        description: 'Deployed application URL'
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    outputs:
      url: ${{ steps.get-url.outputs.url }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET }}
          aws-region: ap-southeast-1

      - name: Deploy
        if: inputs.dry_run == false
        run: |
          echo "Deploying ${{ inputs.image_tag }} to ${{ inputs.environment }}"
          ./deploy.sh ${{ inputs.environment }} ${{ inputs.image_tag }}

      - name: Get deploy URL
        id: get-url
        run: echo "url=https://${{ inputs.environment }}.myapp.com" >> $GITHUB_OUTPUT
```

```yaml
# ─────────────────────────────────────────────────────
# File: .github/workflows/main.yml  (caller)
# ─────────────────────────────────────────────────────
on:
  push:
    branches: [main, develop]

jobs:
  # ── Staging deploy ──
  staging:
    uses: ./.github/workflows/deploy-template.yml    # local file
    # অথবা অন্য repo থেকে:
    # uses: my-org/.github/.github/workflows/deploy-template.yml@main
    with:
      environment: staging
      image_tag: ${{ github.sha }}
    secrets:
      AWS_KEY: ${{ secrets.AWS_KEY }}
      AWS_SECRET: ${{ secrets.AWS_SECRET }}
      # অথবা সব secrets একসাথে:
      # secrets: inherit

  # ── Production deploy ──
  production:
    needs: staging
    if: github.ref == 'refs/heads/main'
    uses: ./.github/workflows/deploy-template.yml
    with:
      environment: production
      image_tag: ${{ github.sha }}
    secrets: inherit

  # ── Output use করো ──
  notify:
    needs: production
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Deployed to: ${{ needs.production.outputs.deploy_url }}"
```

---

## ১৭. Composite Actions — নিজের Action বানাও

### কেন দরকার?
একই ৫টা step বারবার সব workflow এ লিখছো?
এটা একটা custom action এ wrap করো।
একটা জায়গা থেকে maintain করো।

```yaml
# ─────────────────────────────────────────────────────
# File: .github/actions/setup-node-app/action.yml
# এটা composite action
# ─────────────────────────────────────────────────────
name: 'Setup Node App'
description: 'Node setup + cache + install + env setup'

inputs:
  node-version:
    description: 'Node.js version to use'
    required: false
    default: '18'
  install-command:
    description: 'Install command (npm ci or npm install)'
    required: false
    default: 'npm ci'
  working-directory:
    description: 'Working directory'
    required: false
    default: '.'

outputs:
  cache-hit:
    description: 'Whether npm cache was hit'
    value: ${{ steps.cache.outputs.cache-hit }}
  node-version:
    description: 'Actual node version used'
    value: ${{ steps.setup.outputs.node-version }}

runs:
  using: composite           # ← composite = multiple steps
  steps:
    - name: Setup Node.js
      id: setup
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Cache node modules
      id: cache
      uses: actions/cache@v4
      with:
        path: ${{ inputs.working-directory }}/node_modules
        key: node-${{ runner.os }}-${{ inputs.node-version }}-${{ hashFiles(format('{0}/package-lock.json', inputs.working-directory)) }}

    - name: Install dependencies
      shell: bash              # composite action এ shell specify করতে হয়
      working-directory: ${{ inputs.working-directory }}
      run: |
        if [ "${{ steps.cache.outputs.cache-hit }}" != 'true' ]; then
          ${{ inputs.install-command }}
        else
          echo "✅ Using cached node_modules"
        fi
```

```yaml
# ─────────────────────────────────────────────────────
# যেকোনো workflow এ use করো:
# ─────────────────────────────────────────────────────
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # ৫ লাইনের কাজ → ১ লাইনে!
      - name: Setup Node App
        uses: ./.github/actions/setup-node-app    # local action
        id: setup
        with:
          node-version: '20'
          working-directory: './frontend'

      - run: |
          echo "Cache hit: ${{ steps.setup.outputs.cache-hit }}"
          npm run build
```

---

## ১৮. Scheduled Jobs (Cron)

### কেন দরকার?
- রাতে database backup নাও
- প্রতি সোমবার dependency update check করো
- প্রতি ঘণ্টায় external API health check করো
- মাসের শেষে report generate করো

```yaml
on:
  schedule:
    # UTC timezone এ
    - cron: '0 2 * * *'      # প্রতিদিন রাত ২টায়
    - cron: '0 9 * * 1'      # প্রতি সোমবার সকাল ৯টায়

# Cron syntax:
# ┌───── minute (0-59)
# │ ┌───── hour (0-23)
# │ │ ┌───── day of month (1-31)
# │ │ │ ┌───── month (1-12)
# │ │ │ │ ┌───── day of week (0=Sun, 6=Sat)
# │ │ │ │ │
# * * * * *

# Common patterns:
# 0 0 * * *      প্রতিদিন রাত ১২টায়
# 0 */6 * * *    প্রতি ৬ ঘণ্টায়
# 0 9 * * 1-5    সোম-শুক্র সকাল ৯টায়
# 0 0 1 * *      প্রতি মাসের ১ তারিখে
# */30 * * * *   প্রতি ৩০ মিনিটে

jobs:
  # ── Database Backup ──
  backup:
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule'    # শুধু scheduled run এ
    steps:
      - name: Backup PostgreSQL to S3
        env:
          DB_URL: ${{ secrets.DATABASE_URL }}
        run: |
          DATE=$(date +%Y%m%d_%H%M%S)
          pg_dump $DB_URL > backup_${DATE}.sql
          gzip backup_${DATE}.sql
          aws s3 cp backup_${DATE}.sql.gz \
            s3://my-backups/postgres/${DATE}/
          echo "✅ Backup completed: backup_${DATE}.sql.gz"

  # ── Dependency Update Check ──
  check-updates:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
      - run: npm install
      - name: Check for outdated packages
        run: |
          OUTDATED=$(npm outdated --json 2>/dev/null || echo "{}")
          if [ "$OUTDATED" != "{}" ]; then
            echo "⚠️ Outdated packages found!"
            echo $OUTDATED
            # Slack এ notify করো
            curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
              -H 'Content-type: application/json' \
              -d "{\"text\":\"📦 Outdated packages in flight-engine!\"}"
          fi

  # ── Health Check ──
  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Check production health
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
                   https://api.myapp.com/health)

          if [ "$STATUS" != "200" ]; then
            echo "❌ Health check failed! Status: $STATUS"
            # PagerDuty alert পাঠাও
            curl -X POST ${{ secrets.PAGERDUTY_URL }} \
              -d '{"message": "Production is down!"}'
            exit 1
          fi
          echo "✅ Production is healthy (HTTP $STATUS)"
```

---

## ১৯. GitHub Script (API Call)

### কেন দরকার?
PR এ auto comment করতে।
Issue তৈরি করতে।
Label যোগ করতে।
GitHub API এর যেকোনো কাজ করতে।

```yaml
jobs:
  pr-automation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        id: test
        run: npm test
        continue-on-error: true

      # ── PR এ comment করো ──
      - name: Comment test results on PR
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            const status = '${{ steps.test.outcome }}' === 'success'
              ? '✅ All tests passed!'
              : '❌ Tests failed!';

            // GitHub API call — JavaScript দিয়ে!
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## Test Results\n\n${status}\n\nCommit: ${context.sha}`
            });

      # ── Label যোগ করো ──
      - name: Add size label to PR
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            // PR এর changed files count
            const { data: files } = await github.rest.pulls.listFiles({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
            });

            const size = files.length < 10 ? 'small'
                       : files.length < 50 ? 'medium' : 'large';

            await github.rest.issues.addLabels({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              labels: [`size/${size}`]
            });

      # ── Issue তৈরি করো (scheduled job এ কাজে লাগে) ──
      - name: Create weekly report issue
        uses: actions/github-script@v7
        if: github.event_name == 'schedule'
        with:
          script: |
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `Weekly Report - ${new Date().toDateString()}`,
              body: '## Weekly Status\n\n- [ ] Review metrics\n- [ ] Update dependencies',
              labels: ['weekly-report'],
              assignees: ['team-lead']
            });
```

---

## ২০. Self-hosted Runner (বিস্তারিত)

### কেন দরকার?
```
GitHub runner যথেষ্ট না হলে:
  → Private database access দরকার
  → GPU দরকার (ML model training)
  → Faster machine দরকার
  → Cost বাঁচাতে চাও
  → Specific software pre-install রাখতে চাও
```

```bash
# ── Setup on Linux server ──

# ১. GitHub → Settings → Actions → Runners → New self-hosted runner

# ২. Software নামাও
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
  "https://github.com/actions/runner/releases/download/v2.311.0/\
   actions-runner-linux-x64-2.311.0.tar.gz"
tar xzf ./actions-runner-linux-x64.tar.gz

# ৩. Register করো (GitHub থেকে token নাও)
./config.sh \
  --url https://github.com/YOUR_ORG/YOUR_REPO \
  --token YOUR_REGISTRATION_TOKEN \
  --name "my-server" \
  --labels "linux,gpu,high-memory" \   # custom labels
  --work "/tmp/actions"

# ৪. Service হিসেবে চালাও (server restart এও চলবে)
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status   # চলছে কিনা দেখো
```

```yaml
# Workflow এ use করো:
jobs:
  ml-training:
    runs-on: [self-hosted, gpu]    # GPU label এর runner এ চলবে
    steps:
      - uses: actions/checkout@v4
      - run: python train_model.py

  private-db-test:
    runs-on: [self-hosted, linux]  # Private network এ access আছে
    steps:
      - run: |
          # Private database access করো
          psql postgresql://private-db.internal/mydb -c "SELECT 1"
```

---

# 🔴 EXPERT

---

## ২১. OIDC Authentication — Password ছাড়া AWS Login

### কেন দরকার?
```
আগের পদ্ধতি (risky!):
  AWS_ACCESS_KEY_ID → GitHub Secrets এ store
  AWS_SECRET_ACCESS_KEY → GitHub Secrets এ store
  এই keys expire হয় না, যদি leak হয় → বিপদ!

OIDC পদ্ধতি (safe!):
  GitHub → AWS কে বলে "আমি GitHub Actions"
  AWS → trust করলে temporary token দেয় (১ ঘণ্টার জন্য)
  Long-lived secret একদম নেই!
```

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest

    # OIDC এর জন্য permission দরকার
    permissions:
      id-token: write    # OIDC token generate করার permission
      contents: read

    steps:
      - uses: actions/checkout@v4

      # Password ছাড়াই AWS login!
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ap-southeast-1
          # কোনো access key লাগছে না!

      - name: Deploy to AWS
        run: |
          aws ecr get-login-password | docker login ...
          docker push ...
```

```
AWS Console এ করতে হবে:
  IAM → Identity providers → Add provider
    Provider type: OpenID Connect
    Provider URL: https://token.actions.githubusercontent.com
    Audience: sts.amazonaws.com

  IAM → Roles → Create role
    Trusted entity: Web identity
    Identity provider: token.actions.githubusercontent.com
    Condition:
      token.actions.githubusercontent.com:sub = repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main
    Permissions: AmazonECR... (যা দরকার)
```

---

## ২২. Security Hardening

### কেন দরকার?
GitHub Actions এ security hole থাকলে:
- Secret চুরি হতে পারে
- Malicious code run হতে পারে
- Supply chain attack হতে পারে

```yaml
# ── ১. Minimum permissions দাও ──
permissions:
  contents: read      # শুধু code read করতে পারবে
  packages: write     # package push করতে পারবে
  # বাকি সব বন্ধ (actions, deployments, issues, etc.)

# Default সব permission বন্ধ করো:
# Settings → Actions → General → Workflow permissions
# → "Read repository contents and packages permissions"

# ── ২. Action version SHA দিয়ে pin করো ──
steps:
  # ❌ Tag use করো না (tag change হতে পারে)
  - uses: actions/checkout@v4

  # ✅ SHA দিয়ে pin করো (immutable)
  - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
  # v4 এর exact commit SHA — কেউ change করতে পারবে না

  # Dependabot automatically এই SHA update করে দেবে!

# ── ৩. Third-party action সাবধানে use করো ──
# Trusted:
#   actions/*          → GitHub official ✅
#   aws-actions/*      → AWS official ✅
#   docker/*           → Docker official ✅
#   google-github-actions/* → Google official ✅
# সাবধান:
#   Random GitHub user → code review করো আগে ⚠️

# ── ৪. Secret কখনো print করো না ──
steps:
  # ❌ এটা করো না
  - run: echo "Token=${{ secrets.MY_TOKEN }}"

  # ✅ এটা করো
  - run: echo "Token is set=${{ secrets.MY_TOKEN != '' }}"

# ── ৫. Input sanitize করো ──
# PR থেকে আসা input এ malicious code থাকতে পারে!
steps:
  # ❌ Dangerous (script injection!)
  - run: echo "${{ github.event.pull_request.title }}"

  # ✅ Safe (environment variable দিয়ে)
  - env:
      PR_TITLE: ${{ github.event.pull_request.title }}
    run: echo "$PR_TITLE"    # shell injection হবে না

# ── ৬. Pull request এ fork থেকে সাবধান ──
# Fork এর PR → secrets access নেই (GitHub automatically block করে)
# কিন্তু: pull_request_target → secrets আছে → সাবধান!
```

---

## ২৩. Code Scanning (SAST)

### কেন দরকার?
Code এ security vulnerability আগে থেকে ধরো।
Production এ যাওয়ার আগেই fix করো।

```yaml
# ── CodeQL (GitHub built-in) ──
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'    # সোমবার রাত ২টায়

jobs:
  codeql:
    runs-on: ubuntu-latest
    permissions:
      security-events: write    # security alert লিখতে
      actions: read
      contents: read

    strategy:
      matrix:
        language: [javascript, python]   # তোমার project এর languages

    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Analyze
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:${{ matrix.language }}"

  # ── Dependency vulnerability scan ──
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # PR এ vulnerable dependency block করো
      - name: Dependency Review
        uses: actions/dependency-review-action@v4
        if: github.event_name == 'pull_request'
        with:
          fail-on-severity: high      # high severity হলে fail
          deny-licenses: GPL-2.0      # এই license allow না

      # Trivy দিয়ে Docker image scan
      - name: Scan Docker image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'HIGH,CRITICAL'   # শুধু high/critical দেখাও

      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
```

---

## ২৪. Cost Optimization

### কেন দরকার?
GitHub Actions এর cost কমাতে।
Build time কমাতে।
Unnecessary job চলা বন্ধ করতে।

```yaml
# ── ১. Timeout দাও (ঝুলে থাকা job বন্ধ করো) ──
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30      # ৩০ মিনিটে শেষ না হলে cancel
    steps:
      - run: npm run build
        timeout-minutes: 10  # step level timeout

# ── ২. Path filter দাও (অপ্রয়োজনীয় trigger বন্ধ করো) ──
on:
  push:
    paths:
      - 'src/**'         # src/ বদলালেই চালাও
      - 'package*.json'
    paths-ignore:
      - '**.md'          # docs বদলালে চালাবে না
      - '.gitignore'
      - 'docs/**'

# ── ৩. Concurrency দিয়ে duplicate cancel করো ──
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# ── ৪. Cheaper runner use করো ──
jobs:
  lint:
    runs-on: ubuntu-latest    # 2-core, cheap
    # ❌ ubuntu-latest-8-cores না (lint এ 8-core দরকার নেই)

  heavy-build:
    runs-on: ubuntu-latest-4-cores   # শুধু যখন দরকার

# ── ৫. Cache properly করো ──
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      ~/.cache/pip
      .next/cache          # Next.js build cache
    key: build-${{ runner.os }}-${{ hashFiles('package-lock.json') }}

# ── ৬. Skip CI (commit message দিয়ে) ──
# Commit message এ এটা লিখলে workflow চলবে না:
# [skip ci] বা [ci skip]

# ── ৭. Conditional job (changed files check করো) ──
jobs:
  check-changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.filter.outputs.frontend }}
      backend: ${{ steps.filter.outputs.backend }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            frontend:
              - 'frontend/**'
            backend:
              - 'backend/**'

  build-frontend:
    needs: check-changes
    if: needs.check-changes.outputs.frontend == 'true'  # frontend বদলালেই
    runs-on: ubuntu-latest
    steps:
      - run: cd frontend && npm run build

  build-backend:
    needs: check-changes
    if: needs.check-changes.outputs.backend == 'true'   # backend বদলালেই
    runs-on: ubuntu-latest
    steps:
      - run: cd backend && npm run build
```

---

## ২৫. Debugging Techniques

### কেন দরকার?
Workflow fail করলে কী হলো বুঝতে হবে।
Local এ test করার উপায় নেই সবসময়।

```yaml
jobs:
  debug:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # ── ১. সব context print করো ──
      - name: Dump GitHub context
        run: |
          echo "github context:"
          echo '${{ toJSON(github) }}'
          echo "env context:"
          echo '${{ toJSON(env) }}'
          echo "job context:"
          echo '${{ toJSON(job) }}'

      # ── ২. Step debug info ──
      - name: Debug step
        run: |
          echo "Current directory: $(pwd)"
          echo "Files:"
          ls -la
          echo "Environment variables:"
          env | sort
          echo "Docker images:"
          docker images

      # ── ৩. Fail হলে SSH দিয়ে ঢোকো (live debug!) ──
      - name: Setup tmate session
        uses: mxschmitt/action-tmate@v3
        if: failure()           # fail হলে SSH session খোলো
        timeout-minutes: 15     # ১৫ মিনিট পরে close হবে
        with:
          limit-access-to-actor: true  # শুধু তুমি ঢুকতে পারবে

      # ── ৪. Verbose mode চালু করো ──
      - name: Verbose npm install
        run: npm install --verbose
        env:
          ACTIONS_RUNNER_DEBUG: true     # runner debug log
          ACTIONS_STEP_DEBUG: true       # step debug log

      # ── ৫. Artifact হিসেবে log save করো ──
      - name: Upload debug logs
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: debug-logs
          path: |
            /tmp/*.log
            ~/.npm/_logs/
```

### Local এ test করো:

```bash
# act দিয়ে local এ GitHub Actions চালাও
# install: brew install act (macOS)

# সব workflow চালাও
act

# নির্দিষ্ট job চালাও
act -j build

# নির্দিষ্ট event দিয়ে চালাও
act push
act pull_request

# Secret দিয়ে চালাও
act -s MY_SECRET=value

# Dry run (চালাবে না, শুধু দেখাবে)
act -n
```

---

## Quick Reference — সব topics এক নজরে

```
🟢 Foundation:
  Workflow structure    → .yml এর কাঠামো
  Triggers (on:)        → কখন চালু হবে
  Jobs & Steps          → কাজের ভাগ
  Runners               → কোথায় চলে
  Secrets & Variables   → password লুকানো
  Expressions           → dynamic values (${{ }})
  Conditional (if:)     → শর্ত দিয়ে control

🔵 Intermediate:
  Job dependencies      → needs: দিয়ে order
  Job outputs           → jobs এর মধ্যে data share
  Artifacts             → file share করা
  Cache                 → build দ্রুত করো
  Matrix strategy       → parallel testing
  Environments          → approval gate
  Concurrency           → duplicate cancel
  Manual trigger        → button দিয়ে deploy

🟡 Advanced:
  Reusable workflows    → template বানাও
  Composite actions     → custom action
  Scheduled jobs        → cron দিয়ে auto
  GitHub Script         → API call
  Self-hosted runner    → নিজের server

🔴 Expert:
  OIDC authentication   → password ছাড়া AWS
  Security hardening    → safe workflow
  Code scanning         → vulnerability detect
  Cost optimization     → bill কমাও
  Debugging             → problem fix করো
```
