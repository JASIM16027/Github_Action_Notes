
# **Automating Workflows: Making Development Smoother**  

## এই GitHub Actions Workflow — 🔍

---

### প্রথমে পুরো ছবিটা বুঝি

```
কেউ code push করলো
        ↓
Docker image build হলো
        ↓
AWS ECR-এ image push হলো
        ↓
GitOps repo update হলো (Kubernetes জানলো নতুন image আছে)
        ↓
AWS CodeCommit-এ backup হলো
        ↓
Slack-এ notification গেলো ✅ বা ❌
```

---

## ১. TRIGGER — কখন চালু হবে?

```yaml
on:
  push:
    branches:
      - develop
      - master
      - staging
```

> এই তিনটা branch-এ কেউ push করলেই workflow চালু হবে।

```
develop  → development environment
staging  → staging environment  
master   → production environment
```

---

## ২. JOB DEFINE

```yaml
jobs:
  build:
    name: Build, GitOps and Slack Notification
    runs-on: ubuntu-latest
```

> `ubuntu-latest` machine-এ সব কাজ হবে। GitHub একটা fresh Linux server দেবে।

---

## ৩. ENVIRONMENT VARIABLES

```yaml
env:
  ST_ECR_REPOSITORY: ${{ vars.ST_ECR_REPOSITORY }}
```
> AWS ECR-এর URL। যেমন: `123456789.dkr.ecr.ap-southeast-1.amazonaws.com`

```yaml
  ST_CICD_USER: ${{ vars.ST_CICD_USER }}
  ST_CICD_TOKEN: ${{ secrets.ST_CICD_TOKEN }}
```
> GitOps repo-তে push করার জন্য GitHub username ও token।

```yaml
  ST_AWS_ACCESS_KEY_ID: ${{ vars.ST_AWS_ACCESS_KEY_ID }}
  ST_AWS_SECRET_ACCESS_KEY: ${{ secrets.ST_AWS_SECRET_ACCESS_KEY }}
  ST_AWS_REGION: ${{ vars.ST_AWS_REGION }}
```
> AWS-এ login করার credentials। Region যেমন: `ap-southeast-1`

```yaml
  SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```
> Slack-এ notification পাঠানোর URL।

> 💡 `vars.*` = public config, `secrets.*` = sensitive/hidden data

---

## ৪. STEP 1 — Code নামাও

```yaml
- name: Checkout
  uses: actions/checkout@v1
```

> GitHub runner-এ তোমার repository-র code নামিয়ে আনো।
> এটা ছাড়া পরের কোনো কাজই হবে না।

---

## ৫. STEP 2 — AWS-এ Login

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ env.ST_AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ env.ST_AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ env.ST_AWS_REGION }}
```

> AWS SDK কে বলছো — "এই key দিয়ে login করো"।
> পরের সব AWS command এই credentials use করবে।

---

## ৬. STEP 3 — ECR-এ Login

```yaml
- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@v1
```

> ECR = AWS-এর private Docker registry।
> Docker কে বলছো — "এই ECR-এ image push করার permission দাও"।

---

## ৭. STEP 4 — Docker Build & Push (সবচেয়ে গুরুত্বপূর্ণ!)

```yaml
- name: Build, tag, and push image to Amazon ECR
  run: |
    set -e
```
> `set -e` মানে — যেকোনো error হলে সাথে সাথে বন্ধ হয়ে যাও।

```bash
    declare -A branches=( 
      ["refs/heads/develop"]="development"
      ["refs/heads/staging"]="staging"
      ["refs/heads/master"]="production"
    )
```
> একটা map তৈরি করছে:
> ```
> develop branch → "development"
> staging branch → "staging"
> master branch  → "production"
> ```

```bash
    PROJECTS=("flight-engine-b2c" "flight-engine-b2b")
```
> দুইটা project আছে — b2c (Business to Customer) এবং b2b (Business to Business)।
> দুইটার জন্যই আলাদা image build হবে।

```bash
    for branch in "${!branches[@]}"; do
      if [ "${GITHUB_REF}" == "$branch" ]; then
        ENVIRONMENT=${branches[$branch]}
        break
      fi
    done
```
> কোন branch থেকে push এলো সেটা দেখে ENVIRONMENT ঠিক করছে।
> যেমন: `develop` থেকে push → `ENVIRONMENT=development`

```bash
    for PROJECT in "${PROJECTS[@]}"; do
      ECR_REPO="${ST_ECR_REPOSITORY}/${ENVIRONMENT}/${PROJECT}"
```
> প্রতিটা project-এর জন্য ECR path বানাচ্ছে।
> যেমন: `123456.ecr.aws/development/flight-engine-b2c`

```bash
      if ! aws ecr describe-repositories --repository-names "${ENVIRONMENT}/${PROJECT}" > /dev/null 2>&1; then
        aws ecr create-repository --repository-name "${ENVIRONMENT}/${PROJECT}"
      fi
```
> ECR-এ repository আছে কিনা check করো।
> না থাকলে নতুন বানাও। (Auto-create!)

```bash
      docker build -t $ECR_REPO:${{ github.sha }} -f Dockerfile.${ENVIRONMENT} .
```
> Docker image build করো।
> - Tag হিসেবে `github.sha` use করছে (commit hash, যেমন: `a3f8c12`)
> - `Dockerfile.development` বা `Dockerfile.production` আলাদা file আছে

```bash
      docker push $ECR_REPO:${{ github.sha }}
```
> Build করা image AWS ECR-এ push করো।

---

## ৮. STEP 5 — GitOps Update (Kubernetes কে জানাও)

```yaml
- name: Update GitOps repository
```

> এটা হলো **GitOps pattern** — Kubernetes কে directly বলছো না।
> বরং একটা git repo update করছো, Kubernetes সেটা দেখে নিজেই update নেয়।

```bash
    REPO_URL="https://$ST_CICD_USER:$ST_CICD_TOKEN@github.com/..."
```
> GitOps repo-র URL বানাচ্ছে credentials সহ।

```bash
    case ${ENVIRONMENT} in
      development) REPO_URL="...gitops-development.git" ;;
      staging)     REPO_URL="...gitops-staging.git" ;;
      production)  REPO_URL="...cicd.git" ;;
    esac
```
> Environment অনুযায়ী আলাদা GitOps repo:
> ```
> development → gitops-development repo
> staging     → gitops-staging repo
> production  → cicd repo
> ```

```bash
    git clone "$REPO_URL" "$CLONE_DIR"
```
> GitOps repository নামিয়ে আনো।

```bash
    curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
    kustomize edit set image ${PROJECT}=$ST_ECR_REPOSITORY/${ENVIRONMENT}/${PROJECT}:${IMAGE_TAG}
```
> **Kustomize** = Kubernetes config update করার tool।
> নতুন image tag দিয়ে Kubernetes deployment file update করছে।
> এটাই বলছে Kubernetes-কে — "এই নতুন image use করো!"

```bash
    git commit -am "[Github] updating ${ENVIRONMENT} images to ${IMAGE_TAG}"
    git push origin main
```
> Update commit করে push করো।
> Kubernetes (ArgoCD/FluxCD) এই change দেখে automatically deploy করবে।

---

## ৯. STEP 6 — AWS CodeCommit Backup

```yaml
- name: BACKUP CODE to AWS CODE-COMMIT
  uses: pixta-dev/repository-mirroring-action@v1
  if: github.ref == 'refs/heads/master'
```

> শুধু **master branch**-এ push হলে চলবে।
> পুরো repository AWS CodeCommit-এ mirror/backup করছে।
> কারণ: GitHub down হলেও code যেন থাকে!

```yaml
  with:
    target_repo_url: ssh://git-codecommit.us-east-1.amazonaws.com/...
    ssh_private_key: ${{ secrets.CODECOMMITSSHKEY }}
```
> SSH key দিয়ে AWS CodeCommit-এ connect করছে।

---

## ১০. STEP 7 — Slack Notification

```yaml
- uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    fields: repo,message,commit,author,action,eventName,ref,workflow,job,took
  if: always()
```

> `if: always()` — সফল হোক বা fail হোক, **সবসময়** Slack-এ message যাবে।
> Slack-এ যা দেখাবে:
> ```
> ✅ Build successful!
> Repo: flight-engine
> Commit: a3f8c12
> Author: John
> Time taken: 4m 32s
> ```

---

## পুরো Flow এক নজরে

```
git push (develop/staging/master)
          │
          ▼
    Code Checkout
          │
          ▼
    AWS Login & ECR Login
          │
          ▼
    Docker Build
    (b2c + b2b দুইটাই)
          │
          ▼
    ECR-এ Image Push
    (sha tag সহ)
          │
          ▼
    GitOps Repo Update
    (Kustomize দিয়ে)
          │
          ▼
    Kubernetes দেখে
    নতুন image deploy করে
          │
          ├── master হলে → CodeCommit Backup
          │
          ▼
    Slack Notification
    (success বা failure)
```

---

## এই workflow-এ কোন কোন best practice আছে?

| Practice | কোথায় |
|---|---|
| Secret আলাদা রাখা | `secrets.*` use |
| Per-environment আলাদা config | branches map |
| Immutable image tag | `github.sha` use |
| Auto ECR repo create | describe → create |
| GitOps pattern | Kustomize update |
| Backup strategy | CodeCommit mirror |
| Always notify | `if: always()` |


<img width="991" height="465" alt="image" src="https://github.com/user-attachments/assets/2c6b6a6c-0298-448b-8b2d-d6a1a9ead7d7" />

Automating different parts of the development process can save time, improve quality, and reduce errors. Here’s how automation helps:  

1. **Welcoming New Contributors**  
   - Automatically greet new developers, assign issues, and share useful documentation to help them get started.  

2. **Code Quality & Style Checks**  
   - Use tools like **ESLint, Prettier, or SonarQube** to check if the code follows best practices and is free from obvious issues.  

3. **Publishing Docker Images / Packages**  
   - Automatically build and upload **Docker images or software packages** to platforms like **Docker Hub, npm, or GitHub Packages** whenever code is updated.  

4. **Security Scanning**  
   - Detect security vulnerabilities in dependencies and code using tools like **Snyk, Dependabot, or Trivy** before they become a problem.  

5. **Documentation Generation**  
   - Generate API documentation automatically from the code using tools like **Swagger, JSDoc, or Docusaurus**, ensuring it stays up to date.  

6. **Notification & Communication**  
   - Send updates to **Slack, Discord, or email** about build failures, deployments, or other key events.  

7. **Scheduled Tasks**  
   - Automate routine tasks like **backups, report generation, and maintenance** using cron jobs or serverless functions.  

8. **Cross-Platform Testing**  
   - Ensure the app works on **Windows, macOS, and Linux** by running automated tests in different environments.  

### **Why Automate?**  
- **Saves time** by reducing manual work.  
- **Improves reliability** by catching issues early.  
- **Boosts developer productivity** by streamlining workflows.  


Here’s an example of automating some of these workflows using GitHub Actions:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  lint-and-test:
    name: Lint & Test Code
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Install Dependencies
        run: npm install

      - name: Run Linter
        run: npm run lint

      - name: Run Tests
        run: npm test

  build-and-publish:
    name: Build & Publish Docker Image
    runs-on: ubuntu-latest
    needs: lint-and-test
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build & Push Docker Image
        run: |
          docker build -t my-app:latest .
          docker tag my-app:latest my-dockerhub-username/my-app:latest
          docker push my-dockerhub-username/my-app:latest

  notify:
    name: Notify Team
    runs-on: ubuntu-latest
    needs: build-and-publish
    steps:
      - name: Send Notification
        run: |
          curl -X POST -H 'Content-type: application/json' --data \
          '{"text":"🚀 New deployment completed successfully!"}' \
          ${{ secrets.SLACK_WEBHOOK_URL }}
```

### **How This Works**
1. **Lint & Test Code**: Runs ESLint and unit tests on every push or pull request.  
2. **Build & Publish Docker Image**: Builds a Docker image and pushes it to Docker Hub.  
3. **Notify Team**: Sends a message to Slack when deployment is successful.  



# GitHub Actions workflow:

### **Workflow Overview**
- Trigger: Runs on `push` to `develop`, `master`, or `staging` branches.
- Jobs: 
  - Build Docker images
  - Push to Amazon ECR
  - Update GitOps repository
  - Backup code to AWS CodeCommit (for master branch only)
  - Send Slack notifications
  
---

## **Step 1: Checkout Code**
```yaml
- name: Checkout
  uses: actions/checkout@v1
```
- **`uses: actions/checkout@v1`** — This GitHub Action checks out the repository's code into the runner environment.  
- Without this, the workflow won’t have access to the project files.  

---

## **Step 2: Configure AWS Credentials**
```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ env.ST_AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ env.ST_AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ env.ST_AWS_REGION }}
```
- **`uses: aws-actions/configure-aws-credentials@v1`** — Configures AWS credentials securely for CLI commands.  
- **`aws-access-key-id`** & **`aws-secret-access-key`** — Injects credentials from GitHub secrets.  
- **`aws-region`** — Specifies the AWS region where ECR repositories are managed.  

Once configured, AWS CLI commands like `aws ecr` will authenticate automatically.

---

## **Step 3: Login to Amazon ECR**
```yaml
- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@v1
```
- This action handles Docker login for ECR.  
- The AWS CLI internally runs this command:

```bash
aws ecr get-login-password --region $ST_AWS_REGION | docker login --username AWS --password-stdin <account_id>.dkr.ecr.<region>.amazonaws.com
```
- This enables subsequent `docker push` commands to authenticate successfully.

---

## **Step 4: Build, Tag, and Push Docker Images**
```yaml
- name: Build, tag, and push image to Amazon ECR
  run: |
    set -e
```

### **`set -e`**
- Ensures the script stops immediately if any command fails, improving reliability.

---

### **Branch Mapping for Environments**
```bash
declare -A branches=( 
  ["refs/heads/develop"]="development"
  ["refs/heads/staging"]="staging"
  ["refs/heads/master"]="production"
)
```
- **`declare -A`** — Declares an **associative array** (key-value pairs) in bash.  
- Each branch maps to its respective environment.

---

### **Identifying the Environment**
```bash
for branch in "${!branches[@]}"; do
  if [ "${GITHUB_REF}" == "$branch" ]; then
    ENVIRONMENT=${branches[$branch]}
    break
  fi
done
```
- **`for branch in "${!branches[@]}"`** — Iterates through all keys in the associative array.  
- **`if [ "${GITHUB_REF}" == "$branch" ]`** — Checks if the current branch matches one of the defined branches.  
- **`ENVIRONMENT=${branches[$branch]}`** — Assigns the corresponding environment name.  
- The loop breaks once a match is found.

---

### **Project and Repository Setup**
```bash
PROJECTS=("flight-private-b2c" "flight-private-b2b")
for PROJECT in "${PROJECTS[@]}"; do
  ECR_REPO="${ST_ECR_REPOSITORY}/${ENVIRONMENT}/${PROJECT}"
```
- **`PROJECTS=("flight-private-b2c" "flight-private-b2b")`** — Defines the two projects for which Docker images will be built.  
- The loop iterates through both project names.  
- **`ECR_REPO="${ST_ECR_REPOSITORY}/${ENVIRONMENT}/${PROJECT}"`** — Dynamically constructs the ECR repository path.

---

### **Create ECR Repository (if missing)**
```bash
if ! aws ecr describe-repositories --repository-names "${ENVIRONMENT}/${PROJECT}" > /dev/null 2>&1; then
  aws ecr create-repository --repository-name "${ENVIRONMENT}/${PROJECT}"
fi
```
- **`aws ecr describe-repositories`** — Checks if the repository exists.  
- **`> /dev/null 2>&1`** — Silences output (suppress success/failure details).  
- **`aws ecr create-repository`** — Creates the repository if it doesn’t already exist.

---

### **Build, Tag, and Push the Image**
```bash
docker build -t $ECR_REPO:${{ github.sha }} -f Dockerfile.${ENVIRONMENT} .
docker tag $ECR_REPO:${{ github.sha }} $ECR_REPO:${{ github.sha }}
docker push $ECR_REPO:${{ github.sha }}
```
- **`docker build`** — Builds the Docker image using the respective Dockerfile.  
  - `-t` specifies the tag name.  
  - `-f Dockerfile.${ENVIRONMENT}` dynamically picks the appropriate Dockerfile for each environment.  
- **`docker tag`** — Ensures the same image reference is applied twice to avoid confusion.  
- **`docker push`** — Uploads the image to ECR.

---

## **Step 5: Update GitOps Repository**
```yaml
- name: Update GitOps repository
```

### **Set Image Tags in GitOps Repository**
```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
chmod +x kustomize
kustomize edit set image ${PROJECT}=$ST_ECR_REPOSITORY/${ENVIRONMENT}/${PROJECT}:${IMAGE_TAG}
rm -rf kustomize
```
- **`curl`** — Downloads the `kustomize` installer.  
- **`chmod +x`** — Grants execute permissions to `kustomize`.  
- **`kustomize edit set image`** — Updates the Kubernetes manifest with the new Docker image tag.  
- **`rm -rf kustomize`** — Cleans up by removing the downloaded `kustomize` binary.

---

### **Commit and Push Changes**
```bash
if [ $(git status --porcelain | wc -l) -eq "0" ]; then
  echo "🟢 Git repo is clean."
else
  git config --global user.email "github-action@github.com"
  git config --global user.name "Github-Action Pipeline"
  git add .
  git commit -am "[Github] updating ${ENVIRONMENT} images to ${IMAGE_TAG}"
  git pull --ff-only
  git push origin main
fi
```
- **`git status --porcelain`** — Checks for unstaged or modified files.  
- **`git config`** — Sets the author details for the commit.  
- **`git commit -am`** — Stages, commits, and adds a message in one command.  
- **`git pull --ff-only`** — Ensures no merge conflicts occur when pulling updates.  
- **`git push origin main`** — Pushes the changes to the remote repository.

---

## **Step 6: Backup Code to AWS CodeCommit**
```yaml
- name: BACKUP CODE to AWS CODE-COMMIT
```
```bash
uses: pixta-dev/repository-mirroring-action@v1
if: github.ref == 'refs/heads/master'
with:
  target_repo_url:
    ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/sharetrip-flight-private
  ssh_private_key:
    ${{ secrets.CODECOMMITSSHKEY }}
  ssh_username:                                 
    ${{ secrets.CODECOMMITSSHKEYID }}
```
- This step mirrors the repository to AWS CodeCommit as a backup.  
- It only runs for the `master` branch.

---

## **Step 7: Slack Notification**
```yaml
- uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    fields: repo,message,commit,author,action,eventName,ref,workflow,job,took
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
  if: always()
```
- Uses the `action-slack` plugin to send a notification to Slack.  
- **`status`** — Captures the status of the job (success/failure).  
- The `if: always()` condition ensures this step runs regardless of the job's outcome.

---

## **Final Thoughts**
✅ The workflow ensures Docker images are consistently built, tagged, and deployed.  
✅ GitOps practices ensure Kubernetes manifests always reference the correct image.  
✅ The Slack notification keeps the team informed about the deployment's status.  
✅ The backup to AWS CodeCommit ensures no critical code is lost.
