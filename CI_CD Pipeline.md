# CI/CD Pipeline — ধাপে ধাপে Technical ব্যাখ্যা

---

## পুরো Picture একনজরে

```
তুমি (Developer)
      │
      │ git push
      ▼
Feature Branch ──── Pull Request ────► Master Branch
                    (Code Review)            │
                                             │ trigger
                                             ▼
                                        Travis CI
                                        ├── Tests run
                                        ├── Lint check
                                        └── Build
                                             │
                                        ✅ pass হলে
                                             │
                                             ▼
                                        AWS Hosting
                                        (Production Live)
```

---

## ধাপ ১ — তুমি Feature Branch এ কাজ করো

### Branch কী জিনিস?

```
Master Branch  ──●──●──●──●──────────────────► (production code)
                        \
Feature Branch           ●──●──●──●──●        (তোমার নতুন কাজ)
```

Master branch হলো **production এর কোড** — যেটা user রা ব্যবহার করছে। সরাসরি master এ কাজ করলে ভুল হলে production ভেঙে যাবে।

তাই আলাদা branch এ কাজ করো।

### Locally কী হয়:

```bash
# master থেকে নতুন branch বানাও
git checkout -b feature/user-login

# কোড লিখলে
# code editor এ কাজ করলে...

# পরিবর্তন দেখো
git status
git diff

# Stage করো
git add .

# Commit করো
git commit -m "feat: add user login with JWT"

# GitHub এ push করো
git push origin feature/user-login
```

### Git এ internally কী হয়:

```
তোমার PC:
  working directory → git add → staging area → git commit → local repo
                                                                  │
                                                             git push
                                                                  │
                                                                  ▼
                                                            GitHub remote repo
                                                            feature/user-login branch
```

---

## ধাপ ২ — Pull Request (Code Review)

### PR কী?

Feature branch এর কোড master এ নেওয়ার **formal request**। Merge করার আগে teammate রা দেখে, ভুল ধরে।

### PR খুললে GitHub এ কী হয়:

```
তুমি PR open করলে
        │
        ▼
GitHub automatically দেখায়:
┌────────────────────────────────────────┐
│ feature/user-login → master            │
│                                        │
│ Changed files: 5                       │
│ + 120 lines added                      │
│ - 30 lines removed                     │
│                                        │
│ Diff:                                  │
│ + const login = async (req, res) => {  │
│ +   const token = jwt.sign(...)        │
│ - // old auth code                     │
└────────────────────────────────────────┘
```

### Teammate যা করে:

```
Code Review:
✅ "Logic ঠিক আছে"
💬 "এই variable name আরো clear করো"
❌ "এখানে SQL injection হতে পারে!"
        │
        ▼
তুমি fix করলে → আবার push করলে
        │
        ▼
Reviewer approve করলো ✅
        │
        ▼
Merge হলো — feature branch এর কোড master এ গেলো
```

### Merge এ internally কী হয়:

```
আগে:
Master  ──●──●──●
                 \
Feature           ●──●──●

Merge এর পরে:
Master  ──●──●──●──────────●  ← merge commit
                 \         /
Feature           ●──●──●
```

---

## ধাপ ৩ — Travis CI Automatically চালু হয়

### কীভাবে Trigger হয়?

```
Master এ merge হলো
        │
        ▼
GitHub → Webhook fire করলো
{
  "event": "push",
  "ref":   "refs/heads/master",
  "commits": [...]
}
        │
        ▼
Travis CI এই webhook receive করলো
        │
        ▼
Fresh VM (Ubuntu) চালু হলো
        │
        ▼
.travis.yml ফাইল পড়লো
        │
        ▼
কাজ শুরু করলো
```

### .travis.yml দেখতে কেমন:

```yaml
# এই ফাইল তোমার project root এ থাকে
language: node_js
node_js:
  - 18

# Database দরকার হলে
services:
  - postgresql

# Environment variables
env:
  - NODE_ENV=test
  - DATABASE_URL=postgres://localhost/testdb

# Dependency install
install:
  - npm ci

# যা যা run হবে
script:
  - npm run lint        # কোড style check
  - npm test            # unit tests
  - npm run build       # production build

# সব pass হলে deploy করো
deploy:
  provider: elasticbeanstalk   # AWS এ deploy
  access_key_id: $AWS_KEY
  secret_access_key: $AWS_SECRET
  region: ap-southeast-1
  app: myapp
  env: myapp-production
  on:
    branch: master             # শুধু master branch এ
```

### Travis CI তে ধাপে ধাপে কী হয়:

```
Fresh Ubuntu VM চালু
        │
        ▼
GitHub থেকে কোড clone
git clone → master branch
        │
        ▼
Dependencies install
npm ci
        │
        ▼
┌───────────────────────────────┐
│         CI Checks             │
│                               │
│  ১. Lint Check                │
│     npm run lint              │
│     কোড style ঠিক আছে?       │
│     semicolon missing?        │
│     unused variable?          │
│     ✅ pass / ❌ fail          │
│                               │
│  ২. Unit Tests                │
│     npm test                  │
│     প্রতিটা function কি       │
│     ঠিকমতো কাজ করছে?         │
│     ✅ 47 passed / ❌ 2 failed │
│                               │
│  ৩. Build                     │
│     npm run build             │
│     production bundle তৈরি   │
│     কোনো compile error?       │
│     ✅ pass / ❌ fail          │
└───────────────────────────────┘
        │
   সব ✅ pass?
   ┌────┴────┐
   ▼         ▼
  হ্যাঁ      না
   │         │
   │         ▼
   │    Deploy বন্ধ
   │    তোমাকে notify
   │    PR block হয়
   │
   ▼
 Deploy চলবে
```

---

## ধাপ ৪ — AWS এ Automatic Deploy

### Travis CI থেকে AWS এ কীভাবে যায়:

```
Travis CI
    │
    │ AWS credentials দিয়ে
    │ API call করে
    ▼
AWS Elastic Beanstalk
    │
    ├── নতুন version upload হলো (S3 তে)
    │
    ├── নতুন EC2 instance চালু হলো
    │   (নতুন কোড নিয়ে)
    │
    ├── Health check pass করলো?
    │   GET /health → 200 ✅
    │
    ├── Load Balancer traffic নতুনতে দিলো
    │
    └── পুরনো instance বন্ধ হলো
```

### Blue-Green Deployment (Zero Downtime):

```
আগে:
Load Balancer → [Blue: পুরনো কোড] ← user রা এখানে

Deploy শুরু:
Load Balancer → [Blue: পুরনো কোড] ← user রা এখানে
                [Green: নতুন কোড] ← test হচ্ছে

Health check pass:
Load Balancer → [Blue: পুরনো কোড] ← traffic নেই
                [Green: নতুন কোড] ← user রা এখানে

শেষে:
Load Balancer → [Green: নতুন কোড] ← user রা
  Blue বন্ধ হলো

কোনো downtime নেই! ✅
```

---

## পুরো Flow — কতক্ষণ লাগে?

```
তুমি git push করলে
        │  ৩০ সেকেন্ড
        ▼
PR open হলো
        │  ৩০ মিনিট (teammate review)
        ▼
Master এ merge হলো
        │  ১০ সেকেন্ড (webhook)
        ▼
Travis CI শুরু হলো
        │
        ├── npm ci          → ২ মিনিট
        ├── lint            → ৩০ সেকেন্ড
        ├── tests           → ২ মিনিট
        └── build           → ১ মিনিট
        │
        │  মোট ~৬ মিনিট
        ▼
AWS Deploy শুরু
        │
        ├── S3 upload       → ১ মিনিট
        ├── EC2 চালু        → ২ মিনিট
        ├── health check    → ১ মিনিট
        └── traffic switch  → ৩০ সেকেন্ড
        │
        │  মোট ~৫ মিনিট
        ▼
Production Live! 🚀

মোট সময়: ~৪০ মিনিট
তোমার কাজ: শুধু git push ও PR open
বাকি সব: Automatic ✅
```

---

## কোনো ধাপে Fail হলে কী হয়?

```
Lint fail হলে:
  → Travis CI stop
  → তোমাকে email
  → PR এ red cross দেখায়
  → Merge block হয়
  → AWS deploy হয় না
  → Production safe থাকে ✅

Test fail হলে:
  → একই ঘটনা
  → কোন test fail সেটা দেখায়
  → তুমি fix করো, আবার push করো
  → আবার CI চলে

AWS deploy fail হলে:
  → পুরনো version চলতে থাকে
  → User কিছু বোঝে না
  → তুমি log দেখে fix করো ✅
```

---

## Travis CI vs GitHub Actions — পার্থক্য কী?

```
Travis CI:                    GitHub Actions:
─────────────────────         ─────────────────────
আলাদা service                GitHub এর ভেতরেই
.travis.yml                   .github/workflows/*.yml
Free tier limited             Public repo তে free
কম flexible                   অনেক বেশি flexible
কম integration                GitHub এর সাথে tight
                              integration

আজকাল বেশিরভাগ team
GitHub Actions use করে।
Travis CI পুরনো হয়ে যাচ্ছে।
```

---

## এক লাইনে মনে রাখো

```
Feature Branch  → isolation এ কাজ করো
Pull Request    → peer review, ভুল ধরো
Travis CI       → machine দিয়ে automatic test
AWS Deploy      → test pass হলেই production এ যাও

প্রতিটা ধাপ একটা safety net —
শেষ ধাপে পৌঁছাতে হলে আগের সব ধাপ
pass করতে হবে।
Bug production এ যাওয়ার সুযোগ নেই। ✅
```

---

এরপর Terraform দিয়ে AWS infrastructure কোড হিসেবে লেখা, অথবা Kubernetes — কোনটা দেখতে চাও?
