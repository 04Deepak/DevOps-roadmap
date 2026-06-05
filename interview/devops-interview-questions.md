# DevOps Interview Questions & Answers

> **Total Questions:** 150+
> **Levels:** Basic | Intermediate | Advanced
> **Topics:** Git, GitHub, Docker, Jenkins, CI/CD, Ansible, Terraform, AWS, Kubernetes, Monitoring, Secrets, Networking, Linux

---

## Scoring Guide for Interviewers

| Level | Target Candidate | Expect |
|-------|-----------------|--------|
| Basic | 0-1 year experience | Should answer 80%+ of basic questions |
| Intermediate | 1-3 years experience | Should answer basic + 70% intermediate |
| Advanced | 3+ years experience | Should answer all levels confidently |

---

# 1. GIT & GITHUB

## Basic

**Q1: What is Git and why is it used?**

> Git is a distributed version control system that tracks changes in source code. It allows multiple developers to work on the same project simultaneously, maintain history of all changes, and revert to any previous version if needed. Every developer has a full copy of the repository on their local machine.

**Q2: What is the difference between Git and GitHub?**

> | Git | GitHub |
> |-----|--------|
> | Version control tool | Cloud hosting platform for Git repos |
> | Installed on local machine | Web-based service |
> | Manages code history | Adds collaboration features (PRs, Issues) |
> | Works offline | Requires internet |
> | Command-line tool | Web UI + API |

**Q3: Explain the basic Git workflow.**

> 1. `git clone` or `git init` — Get/create a repository
> 2. Edit files in working directory
> 3. `git add` — Stage changes (move to staging area)
> 4. `git commit` — Save snapshot to local repository
> 5. `git push` — Upload commits to remote repository
>
> The three areas: **Working Directory → Staging Area → Repository**

**Q4: What is the difference between `git pull` and `git fetch`?**

> | `git fetch` | `git pull` |
> |-------------|------------|
> | Downloads changes from remote | Downloads AND merges changes |
> | Does NOT modify working directory | Updates working directory |
> | Safe — lets you review first | Can cause merge conflicts |
> | `git fetch` + `git merge` = `git pull` | Shortcut for fetch + merge |

**Q5: What is `.gitignore`? Give examples of what you would put in it.**

> `.gitignore` tells Git which files/folders to NOT track. Common entries:
> - `node_modules/` — dependencies (can be reinstalled)
> - `.env` — environment secrets
> - `*.log` — log files
> - `.DS_Store` — OS files
> - `__pycache__/` — Python cache
> - `.terraform/` — Terraform state files
> - `*.tfvars` — Terraform variable files with secrets

**Q6: What is a Git branch?**

> A branch is a separate line of development. It's a pointer to a specific commit. Branches allow you to work on features, bug fixes, or experiments without affecting the main codebase. The default branch is usually called `main` or `master`.

## Intermediate

**Q7: What is the difference between `git merge` and `git rebase`?**

> | `git merge` | `git rebase` |
> |-------------|-------------|
> | Creates a merge commit | Rewrites commit history |
> | Preserves complete history | Creates linear history |
> | Non-destructive | Changes commit SHAs |
> | Safe for shared branches | Only use on local/private branches |
>
> **Rule:** Use merge for shared branches, rebase for local cleanup before merging.

**Q8: What is a Git conflict and how do you resolve it?**

> A conflict occurs when two branches modify the same part of the same file. Git marks the conflict:
> ```
> <<<<<<< HEAD
> Current branch content
> =======
> Incoming branch content
> >>>>>>> feature-branch
> ```
> **Resolution:** Manually edit the file, keep desired changes, remove conflict markers, then `git add` and `git commit`.

**Q9: What is a Pull Request (PR)?**

> A PR is a request to merge code from one branch into another. It enables:
> - **Code review** — teammates review your changes before merging
> - **Discussion** — comments and feedback on specific lines
> - **CI checks** — automated tests run against the PR
> - **Approval workflow** — required approvals before merge
>
> PR is a GitHub/GitLab feature, not a Git feature.

**Q10: Explain GitFlow branching strategy.**

> - `main` — production-ready code, always stable
> - `develop` — integration branch for features
> - `feature/*` — new features, branch from develop
> - `release/*` — prepare for release, branch from develop
> - `hotfix/*` — emergency production fixes, branch from main
>
> Flow: feature → develop → release → main

**Q11: What is `git stash`?**

> `git stash` temporarily saves uncommitted changes and reverts the working directory to a clean state. Useful when you need to switch branches but aren't ready to commit.
> - `git stash` — save changes
> - `git stash list` — view stashed changes
> - `git stash pop` — restore and remove from stash
> - `git stash apply` — restore but keep in stash

**Q12: What is `git cherry-pick`?**

> `git cherry-pick <commit-hash>` applies a specific commit from one branch onto another without merging the entire branch. Used when you need just one specific fix from another branch.

## Advanced

**Q13: What is `git bisect` and when do you use it?**

> `git bisect` performs a binary search through commit history to find which commit introduced a bug. You mark commits as "good" or "bad" and Git narrows down the problematic commit:
> ```bash
> git bisect start
> git bisect bad          # Current commit has the bug
> git bisect good abc123  # This old commit was fine
> # Git checks out middle commit, you test it
> git bisect good/bad     # Repeat until found
> git bisect reset        # Done
> ```

**Q14: How do you recover a deleted branch or lost commit?**

> Use `git reflog` — it records all HEAD movements for 90 days:
> ```bash
> git reflog                           # Find the commit SHA
> git checkout -b recovered abc123     # Recreate the branch
> ```
> Reflog is local only and is the safety net for most Git disasters.

**Q15: What are Git hooks? Give examples.**

> Git hooks are scripts that run automatically at certain Git events:
> - `pre-commit` — run linters/formatters before commit
> - `pre-push` — run tests before pushing
> - `commit-msg` — validate commit message format
> - `post-merge` — install dependencies after merge
>
> Located in `.git/hooks/`. Teams use tools like **Husky** (Node.js) or **pre-commit** (Python) to share hooks across the team.

---

# 2. DOCKER & CONTAINERS

## Basic

**Q16: What is Docker?**

> Docker is a platform for building, running, and shipping applications in containers. A container packages an application with all its dependencies so it runs consistently across any environment.

**Q17: What is the difference between a Docker Image and a Container?**

> | Image | Container |
> |-------|-----------|
> | Blueprint/template | Running instance of an image |
> | Read-only | Read-write |
> | Built from Dockerfile | Created from an image |
> | Can be shared via registry | Lives on the host where it runs |
> | Like a class in programming | Like an object (instance of class) |

**Q18: What is a Dockerfile? Explain common instructions.**

> A Dockerfile is a text file with instructions to build a Docker image:
> - `FROM` — base image (e.g., `python:3.11`)
> - `WORKDIR` — set working directory
> - `COPY` / `ADD` — copy files into image
> - `RUN` — execute commands during build
> - `ENV` — set environment variables
> - `EXPOSE` — document which port the app uses
> - `CMD` — default command when container starts
> - `ENTRYPOINT` — main command (cannot be overridden easily)

**Q19: What is the difference between `CMD` and `ENTRYPOINT`?**

> | CMD | ENTRYPOINT |
> |-----|-----------|
> | Default command, can be overridden | Main command, harder to override |
> | `docker run image <new-cmd>` replaces CMD | `docker run image <args>` appends to ENTRYPOINT |
> | Use for default arguments | Use for the main executable |
>
> Best practice: Use `ENTRYPOINT` for the command and `CMD` for default arguments:
> ```dockerfile
> ENTRYPOINT ["python"]
> CMD ["app.py"]
> ```

**Q20: What is Docker Compose?**

> Docker Compose is a tool for defining and running multi-container applications using a YAML file (`docker-compose.yml`). Instead of running multiple `docker run` commands, you define all services, networks, and volumes in one file and use `docker compose up` to start everything.

## Intermediate

**Q21: What is a multi-stage build and why is it used?**

> Multi-stage builds use multiple `FROM` statements to create smaller production images. Build tools and dependencies stay in the build stage and are NOT included in the final image.
> ```dockerfile
> FROM node:20 AS builder     # Stage 1: Build
> COPY . .
> RUN npm ci && npm run build
>
> FROM node:20-alpine          # Stage 2: Production
> COPY --from=builder /app/dist ./dist
> CMD ["node", "dist/server.js"]
> ```
> **Benefit:** Final image is much smaller (e.g., 450MB → 100MB).

**Q22: What is the difference between `COPY` and `ADD`?**

> | COPY | ADD |
> |------|-----|
> | Simply copies files | Copies files + extra features |
> | Recommended by best practices | Can auto-extract tar archives |
> | Explicit and predictable | Can download from URLs |
>
> **Best practice:** Always use `COPY` unless you specifically need `ADD`'s extra features.

**Q23: How does Docker layer caching work? How do you optimize it?**

> Each Dockerfile instruction creates a layer. Docker caches layers and reuses them if the instruction and content haven't changed.
>
> **Optimization:** Put things that change LEAST at the top, things that change MOST at the bottom:
> ```dockerfile
> COPY package.json .       # Changes rarely → cached
> RUN npm install            # Uses cache if package.json unchanged
> COPY . .                   # Changes often → rebuild from here
> ```

**Q24: What are Docker volumes? Types?**

> Volumes persist data beyond container lifecycle. Three types:
> - **Named volumes** — managed by Docker (`docker volume create data`)
> - **Bind mounts** — map host directory to container (`-v /host/path:/container/path`)
> - **tmpfs mounts** — stored in memory only, not persisted
>
> Use named volumes for databases, bind mounts for development.

**Q25: How do containers communicate with each other?**

> - **Same Docker network:** Containers can reach each other by container name (Docker DNS)
> - **Docker Compose:** All services are on the same default network automatically
> - **Custom network:** `docker network create mynet` then `--network mynet`
> - **Host network:** Container uses host's network stack directly

## Advanced

**Q26: How would you reduce Docker image size?**

> 1. Use **Alpine-based** images (`node:20-alpine` instead of `node:20`)
> 2. Use **multi-stage builds** — separate build and runtime
> 3. Combine `RUN` commands to reduce layers
> 4. Add `.dockerignore` to exclude unnecessary files
> 5. Remove package manager cache (`rm -rf /var/lib/apt/lists/*`)
> 6. Don't install unnecessary packages (`--no-install-recommends`)
> 7. Use `--no-cache-dir` for pip installs

**Q27: How do you handle secrets in Docker?**

> - **Never** put secrets in Dockerfile or image
> - Use **environment variables** at runtime (`docker run -e SECRET=value`)
> - Use **Docker secrets** (Swarm mode) — mounted as files in `/run/secrets/`
> - Use **volume mounts** for secret files
> - In production: use **HashiCorp Vault**, AWS Secrets Manager, or Kubernetes Secrets
> - Use **`.env` files** with Docker Compose (but never commit them)

**Q28: What is the difference between `docker stop` and `docker kill`?**

> - `docker stop` — sends SIGTERM (graceful shutdown), waits 10s, then SIGKILL
> - `docker kill` — sends SIGKILL immediately (forced shutdown)
>
> Always prefer `docker stop` to allow the application to clean up (close connections, save state).

---

# 3. JENKINS

## Basic

**Q29: What is Jenkins?**

> Jenkins is an open-source automation server used to build, test, and deploy software. It is the most widely used CI/CD tool. It supports plugins for integrating with virtually any tool in the DevOps ecosystem.

**Q30: What is the difference between a Freestyle Job and a Pipeline?**

> | Freestyle Job | Pipeline |
> |---------------|----------|
> | Configured via UI | Written as code (Jenkinsfile) |
> | Limited flexibility | Full programming capabilities |
> | Hard to version control | Stored in Git with the project |
> | Simple tasks | Complex, multi-stage workflows |
> | No code review for jobs | Jenkinsfile can be reviewed in PRs |

**Q31: What is a Jenkinsfile?**

> A Jenkinsfile is a text file that defines a Jenkins Pipeline as code. It can be:
> - **Declarative** — structured, opinionated syntax (recommended)
> - **Scripted** — full Groovy programming language
>
> Stored in the root of the repository alongside the application code.

**Q32: What is the Master-Agent (Master-Slave) architecture in Jenkins?**

> - **Master (Controller):** Schedules jobs, manages agents, serves UI, stores configuration
> - **Agent (Worker):** Executes the actual build jobs
>
> This distributes workload across multiple machines. Agents can be on different OS, have different tools installed, and run jobs in parallel.

## Intermediate

**Q33: Explain the stages of a Declarative Jenkins Pipeline.**

> ```groovy
> pipeline {
>     agent any                    // Where to run
>     environment {                // Environment variables
>         APP = 'myapp'
>     }
>     stages {
>         stage('Build') {         // Build stage
>             steps { sh 'npm ci' }
>         }
>         stage('Test') {          // Test stage
>             steps { sh 'npm test' }
>         }
>         stage('Deploy') {        // Deploy stage
>             when { branch 'main' }  // Conditional
>             steps { sh './deploy.sh' }
>         }
>     }
>     post {                       // Post-build actions
>         success { echo 'Done!' }
>         failure { echo 'Failed!' }
>     }
> }
> ```

**Q34: What are Jenkins Shared Libraries?**

> Shared Libraries are reusable Groovy code stored in a separate Git repository that can be imported into any Jenkinsfile. They prevent code duplication when multiple projects use similar pipeline logic.
>
> ```groovy
> @Library('my-shared-lib') _
> pipeline {
>     stages {
>         stage('Deploy') {
>             steps { deployApp('myapp', 'production') }  // From shared library
>         }
>     }
> }
> ```

**Q35: How do you handle credentials/secrets in Jenkins?**

> - Store in **Jenkins Credentials Manager** (Manage Jenkins → Credentials)
> - Types: Username/Password, Secret text, SSH key, Certificate
> - Access in pipeline using `withCredentials()`:
> ```groovy
> withCredentials([usernamePassword(credentialsId: 'docker-creds',
>     usernameVariable: 'USER', passwordVariable: 'PASS')]) {
>     sh 'docker login -u $USER -p $PASS'
> }
> ```
> - Never hardcode credentials in Jenkinsfile
> - Never echo/print credentials in logs

**Q36: What are Jenkins build triggers?**

> - **Poll SCM** — Jenkins checks Git for changes on schedule (e.g., every 5 min)
> - **Webhook** — GitHub sends notification to Jenkins on push (recommended)
> - **Upstream job** — trigger after another job completes
> - **Scheduled (cron)** — run at specific times (`H 2 * * *` = daily at 2 AM)
> - **Manual** — triggered by user

## Advanced

**Q37: How do you run parallel stages in Jenkins?**

> ```groovy
> stage('Tests') {
>     parallel {
>         stage('Unit Tests') {
>             steps { sh 'npm run test:unit' }
>         }
>         stage('Integration Tests') {
>             steps { sh 'npm run test:integration' }
>         }
>         stage('Security Scan') {
>             steps { sh 'trivy image myapp' }
>         }
>     }
> }
> ```
> Parallel stages run simultaneously, reducing total pipeline time.

**Q38: How do you implement a multi-branch pipeline?**

> Multi-branch pipeline automatically discovers branches in a repo and creates a pipeline for each. Configure by:
> 1. Create "Multibranch Pipeline" item in Jenkins
> 2. Point to Git repository
> 3. Jenkins scans all branches for Jenkinsfile
> 4. Each branch gets its own pipeline run
> 5. Use `when { branch 'main' }` for branch-specific stages
>
> PRs also get their own pipeline automatically.

**Q39: How do you optimize Jenkins pipeline performance?**

> 1. Use **parallel stages** for independent tasks
> 2. Use **Docker agents** for clean, disposable build environments
> 3. **Cache dependencies** (npm cache, Maven local repo)
> 4. Use **stash/unstash** to pass files between stages
> 5. Use **lightweight checkout** — only checkout needed files
> 6. **Skip stages** with `when` conditions
> 7. Distribute load across **multiple agents**
> 8. Use **incremental builds** — only rebuild what changed

---

# 4. CI/CD

## Basic

**Q40: What is CI/CD?**

> - **CI (Continuous Integration):** Automatically build and test every code change pushed to the repository
> - **CD (Continuous Delivery):** Automatically prepare releases and deploy to staging; production requires manual approval
> - **CD (Continuous Deployment):** Automatically deploy every change to production without manual approval

**Q41: What are the stages of a CI/CD pipeline?**

> 1. **Source** — code push triggers the pipeline
> 2. **Build** — compile code, install dependencies
> 3. **Test** — unit tests, integration tests, security scans
> 4. **Package** — build Docker image, create artifacts
> 5. **Deploy** — deploy to staging/production
> 6. **Monitor** — verify deployment, watch metrics

**Q42: What is a build artifact?**

> A build artifact is the output of the build process — the deployable unit. Examples:
> - Docker image
> - JAR/WAR file (Java)
> - Compiled binary
> - npm package
> - Zip/tar archive
>
> Artifacts are stored in registries (Docker Hub, Nexus, Artifactory, S3).

## Intermediate

**Q43: What is the difference between Continuous Delivery and Continuous Deployment?**

> | Continuous Delivery | Continuous Deployment |
> |--------------------|-----------------------|
> | Every change is ready to deploy | Every change IS deployed automatically |
> | Manual approval for production | No manual approval needed |
> | Human decides when to release | Fully automated to production |
> | Most companies use this | Requires very mature testing |

**Q44: Explain blue-green deployment.**

> Two identical production environments: Blue (current) and Green (new).
> 1. Deploy new version to Green
> 2. Test Green environment
> 3. Switch traffic from Blue to Green (via load balancer/DNS)
> 4. If issues: switch back to Blue instantly (rollback)
> 5. Blue becomes standby for next deployment
>
> **Benefit:** Zero downtime, instant rollback.

**Q45: Explain canary deployment.**

> Roll out new version to a small subset of users first:
> 1. Deploy new version alongside old version
> 2. Route 5-10% traffic to new version
> 3. Monitor error rates and performance
> 4. If healthy: gradually increase (25% → 50% → 100%)
> 5. If issues: route all traffic back to old version
>
> **Benefit:** Limits blast radius of problems.

**Q46: What is a rolling update?**

> Replace instances of the old version one at a time with the new version:
> 1. Start one new pod
> 2. Wait for it to be healthy
> 3. Remove one old pod
> 4. Repeat until all pods are updated
>
> **Kubernetes defaults to this strategy.** Controlled by `maxSurge` and `maxUnavailable`.

## Advanced

**Q47: How do you handle database schema changes in CI/CD?**

> 1. Use **migration tools** (Flyway, Alembic, Knex migrations)
> 2. Migrations must be **backward compatible** — old code must work with new schema
> 3. **Expand-Contract pattern:**
>    - Step 1: Add new column (expand) — deploy app that writes to both old and new
>    - Step 2: Migrate data
>    - Step 3: Remove old column (contract) — deploy app that uses only new
> 4. Never rename or drop columns directly — add new, migrate, then drop old
> 5. Run migrations as a **separate pipeline stage** before deploying new code

**Q48: How do you implement rollback in CI/CD?**

> - **Docker:** Redeploy previous image tag (`myapp:v1.2.3` instead of `myapp:v1.2.4`)
> - **Kubernetes:** `kubectl rollout undo deployment/myapp`
> - **Blue-Green:** Switch load balancer back to old environment
> - **GitOps (ArgoCD):** Revert the Git commit, ArgoCD auto-deploys previous state
> - **Feature Flags:** Disable the feature without redeploying
>
> Best practice: Keep last 3-5 versions available for quick rollback.

---

# 5. ANSIBLE

## Basic

**Q49: What is Ansible?**

> Ansible is an open-source configuration management and automation tool. It automates server setup, application deployment, and task automation. It uses YAML for configuration (playbooks) and is agentless — no software needs to be installed on managed servers (uses SSH).

**Q50: What is the difference between Ansible and Terraform?**

> | Ansible | Terraform |
> |---------|-----------|
> | Configuration management | Infrastructure provisioning |
> | Configures servers (install, configure) | Creates servers (VMs, networks, DBs) |
> | Procedural (step by step) | Declarative (desired state) |
> | Agentless (SSH) | Uses cloud provider APIs |
> | Mutable infrastructure | Immutable infrastructure |
> | Best for: configure servers | Best for: create cloud resources |

**Q51: What is an Ansible Playbook?**

> A playbook is a YAML file that defines a set of tasks to execute on target servers. It specifies which hosts to target, what tasks to run, and in what order:
> ```yaml
> - name: Configure Web Server
>   hosts: webservers
>   become: yes
>   tasks:
>     - name: Install Nginx
>       apt: name=nginx state=present
>     - name: Start Nginx
>       service: name=nginx state=started enabled=yes
> ```

**Q52: What is an Ansible Inventory?**

> An inventory file defines the target servers Ansible manages. It groups servers and sets connection details:
> ```ini
> [webservers]
> web01 ansible_host=192.168.1.10
> web02 ansible_host=192.168.1.11
>
> [dbservers]
> db01 ansible_host=192.168.1.20
>
> [all:vars]
> ansible_user=ubuntu
> ```

## Intermediate

**Q53: What is idempotency in Ansible?**

> Idempotency means running the same playbook multiple times produces the same result without side effects. If Nginx is already installed, Ansible won't reinstall it — it reports "ok" instead of "changed."
>
> This is critical — you should be able to run a playbook 100 times safely.

**Q54: What are Ansible Roles?**

> Roles organize playbooks into reusable, modular components with a standard directory structure:
> ```
> roles/webserver/
> ├── tasks/main.yml      # Tasks to execute
> ├── handlers/main.yml   # Event handlers (e.g., restart service)
> ├── templates/           # Jinja2 templates
> ├── files/               # Static files to copy
> ├── vars/main.yml       # Role variables
> ├── defaults/main.yml   # Default variables (lowest priority)
> └── meta/main.yml       # Role metadata and dependencies
> ```
> Used in playbook: `roles: [webserver, database, monitoring]`

**Q55: What is Ansible Vault?**

> Ansible Vault encrypts sensitive data (passwords, API keys, certificates) inside playbooks or variable files:
> ```bash
> ansible-vault create secrets.yml       # Create encrypted file
> ansible-vault edit secrets.yml         # Edit encrypted file
> ansible-vault encrypt existing.yml     # Encrypt existing file
> ansible-playbook site.yml --ask-vault-pass  # Run with vault
> ```
> Secrets are encrypted with AES-256. The vault password is needed at runtime.

**Q56: What are Handlers in Ansible?**

> Handlers are tasks that run only when notified by another task. They run once at the end, even if notified multiple times:
> ```yaml
> tasks:
>   - name: Update Nginx config
>     template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
>     notify: Restart Nginx    # Triggers handler ONLY if config changed
>
> handlers:
>   - name: Restart Nginx
>     service: name=nginx state=restarted
> ```

## Advanced

**Q57: What is the difference between `include` and `import` in Ansible?**

> | `import_*` (Static) | `include_*` (Dynamic) |
> |---------------------|----------------------|
> | Processed at playbook parse time | Processed at runtime |
> | Cannot use loops or conditionals on import | Can use loops and `when` |
> | Variables resolved at parse time | Variables resolved at execution |
> | Better for fixed, predictable tasks | Better for conditional/dynamic inclusion |

**Q58: How do you handle rolling deployments with Ansible?**

> Use `serial` to deploy to a subset of hosts at a time:
> ```yaml
> - name: Rolling deployment
>   hosts: webservers
>   serial: 2              # Deploy to 2 servers at a time
>   max_fail_percentage: 25  # Stop if 25% of hosts fail
>   tasks:
>     - name: Remove from load balancer
>       # ...
>     - name: Deploy new version
>       # ...
>     - name: Health check
>       uri: url=http://localhost/health status_code=200
>     - name: Add back to load balancer
>       # ...
> ```

---

# 6. TERRAFORM

## Basic

**Q59: What is Terraform?**

> Terraform is an Infrastructure as Code (IaC) tool by HashiCorp. It lets you define cloud infrastructure in declarative configuration files (HCL), version control it, and automatically create/update/destroy resources. It supports multiple cloud providers (AWS, Azure, GCP).

**Q60: What is the Terraform workflow?**

> 1. `terraform init` — Initialize, download provider plugins
> 2. `terraform plan` — Preview what will be created/changed/destroyed
> 3. `terraform apply` — Execute the plan, create real resources
> 4. `terraform destroy` — Remove all managed resources

**Q61: What is Terraform state?**

> The state file (`terraform.tfstate`) is a JSON file that maps your Terraform configuration to real-world resources. It tracks:
> - What resources exist
> - Their current attributes (IDs, IPs)
> - Dependencies between resources
>
> Without state, Terraform wouldn't know what already exists.

**Q62: What is a Terraform Provider?**

> A provider is a plugin that interfaces with cloud platforms or services. Examples:
> - `hashicorp/aws` — manages AWS resources
> - `hashicorp/azurerm` — manages Azure resources
> - `hashicorp/kubernetes` — manages Kubernetes resources
>
> Declared in configuration:
> ```hcl
> provider "aws" {
>   region = "us-east-1"
> }
> ```

## Intermediate

**Q63: What is remote state and why is it important?**

> By default, state is stored locally. Remote state stores it in a shared location (S3, Azure Blob, GCS):
> ```hcl
> backend "s3" {
>   bucket = "my-terraform-state"
>   key    = "prod/terraform.tfstate"
>   region = "us-east-1"
>   dynamodb_table = "terraform-locks"  # State locking
> }
> ```
> **Benefits:**
> - **Team collaboration** — everyone uses the same state
> - **Locking** — prevents two people from modifying simultaneously
> - **Versioning** — S3 versioning to recover old state
> - **Security** — state can contain secrets, centralized access control

**Q64: What are Terraform Modules?**

> Modules are reusable packages of Terraform configuration. Instead of repeating code, define it once and call it multiple times:
> ```hcl
> module "web_server" {
>   source        = "./modules/webserver"
>   instance_type = "t2.micro"
>   name          = "web-01"
> }
>
> module "web_server_2" {
>   source        = "./modules/webserver"
>   instance_type = "t2.small"
>   name          = "web-02"
> }
> ```

**Q65: What is `terraform plan` vs `terraform apply`?**

> | `terraform plan` | `terraform apply` |
> |-----------------|-------------------|
> | Dry run — shows what WOULD happen | Actually creates/changes resources |
> | No changes made | Real infrastructure changes |
> | Review before applying | Executes the plan |
> | Safe to run anytime | Requires confirmation |

**Q66: What is the difference between `variable` and `local` in Terraform?**

> - **Variables** (`variable`) — input values, can be set by users:
>   ```hcl
>   variable "region" { default = "us-east-1" }
>   ```
> - **Locals** (`locals`) — computed values used internally, NOT settable by users:
>   ```hcl
>   locals {
>     name_prefix = "${var.project}-${var.environment}"
>   }
>   ```

## Advanced

**Q67: How do you manage multiple environments (dev/staging/prod) in Terraform?**

> Several approaches:
> 1. **Workspaces:** `terraform workspace new staging` — same code, different state files
> 2. **Directory structure:** Separate directories per environment:
>    ```
>    environments/
>    ├── dev/main.tf
>    ├── staging/main.tf
>    └── prod/main.tf
>    ```
> 3. **Var files:** Same code, different `.tfvars`:
>    ```bash
>    terraform apply -var-file="prod.tfvars"
>    ```
> 4. **Terragrunt:** Wrapper tool that keeps Terraform DRY across environments

**Q68: What is `terraform import`?**

> `terraform import` brings existing infrastructure (created manually or by another tool) under Terraform management:
> ```bash
> terraform import aws_instance.web i-0abc123def
> ```
> This updates the state file but does NOT generate configuration. You must manually write the matching `.tf` code.

**Q69: What is a `data` source in Terraform?**

> Data sources fetch information about existing resources NOT managed by Terraform:
> ```hcl
> data "aws_ami" "ubuntu" {
>   most_recent = true
>   owners      = ["099720109477"]
>   filter {
>     name   = "name"
>     values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-*"]
>   }
> }
>
> resource "aws_instance" "web" {
>   ami = data.aws_ami.ubuntu.id   # Use the result
> }
> ```

**Q70: How do you handle state file corruption or conflicts?**

> 1. Enable **S3 versioning** — recover previous state versions
> 2. Use **DynamoDB locking** — prevents concurrent modifications
> 3. `terraform state pull` — download current state
> 4. `terraform state push` — upload corrected state (dangerous)
> 5. `terraform force-unlock <LOCK_ID>` — release stuck lock
> 6. If corrupted: restore from S3 version history, or `terraform import` resources

---

# 7. AWS

## Basic

**Q71: What is the difference between IaaS, PaaS, and SaaS?**

> | Type | You Manage | Provider Manages | Example |
> |------|-----------|-----------------|---------|
> | **IaaS** | App, Data, Runtime, OS | Hardware, Networking | EC2, S3 |
> | **PaaS** | App, Data | Runtime, OS, Hardware | Elastic Beanstalk, Heroku |
> | **SaaS** | Nothing | Everything | Gmail, Slack, Office 365 |

**Q72: What is EC2?**

> EC2 (Elastic Compute Cloud) provides resizable virtual servers in the cloud. You choose:
> - **AMI** — operating system image
> - **Instance type** — CPU, memory (t2.micro, t3.large)
> - **Key pair** — SSH access
> - **Security group** — firewall rules
> - **Storage** — EBS volumes

**Q73: What is S3?**

> S3 (Simple Storage Service) is object storage for files. Features:
> - Unlimited storage, individual files up to 5TB
> - Organized in buckets (globally unique names)
> - Durability: 99.999999999% (11 nines)
> - Storage classes: Standard, Infrequent Access, Glacier (archival)
> - Use cases: backups, static websites, log storage, data lakes

**Q74: What is a VPC?**

> VPC (Virtual Private Cloud) is your private network in AWS. You control:
> - IP address range (CIDR block)
> - Subnets (public and private)
> - Route tables
> - Internet Gateway (public access)
> - NAT Gateway (private subnet internet access)
> - Security Groups and NACLs (firewall)

**Q75: What is IAM?**

> IAM (Identity and Access Management) controls WHO can access WHAT in AWS:
> - **Users** — individual people
> - **Groups** — collection of users (e.g., "developers")
> - **Roles** — temporary permissions for services (e.g., EC2 accessing S3)
> - **Policies** — JSON documents defining permissions (allow/deny actions on resources)
>
> **Best practice:** Least privilege — give minimum permissions needed.

## Intermediate

**Q76: What is the difference between Security Group and NACL?**

> | Security Group | NACL |
> |---------------|------|
> | Instance level | Subnet level |
> | Stateful (return traffic auto-allowed) | Stateless (must allow both in and out) |
> | Allow rules only | Allow AND deny rules |
> | Evaluated as a whole | Evaluated in order (by rule number) |
> | Applied to ENI | Applied to all instances in subnet |

**Q77: What is an Auto Scaling Group?**

> Auto Scaling automatically adjusts the number of EC2 instances based on demand:
> - **Min:** Minimum instances always running
> - **Max:** Maximum instances allowed
> - **Desired:** Target number of instances
> - **Scaling policies:** Scale based on CPU, memory, requests, schedule
> - Works with ELB to distribute traffic

**Q78: What is the difference between ALB and NLB?**

> | ALB (Application LB) | NLB (Network LB) |
> |----------------------|-------------------|
> | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP) |
> | Path-based routing | Ultra-low latency |
> | Host-based routing | Millions of requests/sec |
> | SSL termination | Static IP/Elastic IP |
> | WebSocket support | Best for non-HTTP traffic |

**Q79: What is CloudFormation? How does it compare to Terraform?**

> CloudFormation is AWS's native IaC service. Comparison:
> | CloudFormation | Terraform |
> |---------------|-----------|
> | AWS only | Multi-cloud |
> | JSON/YAML | HCL |
> | Free (AWS managed) | Open source |
> | Deep AWS integration | Provider plugins |
> | Stacks and StackSets | Workspaces and modules |

## Advanced

**Q80: Explain the AWS Well-Architected Framework pillars.**

> 1. **Operational Excellence** — run and monitor systems, automate changes
> 2. **Security** — protect data, systems, and assets
> 3. **Reliability** — recover from failures, handle demand
> 4. **Performance Efficiency** — use computing resources efficiently
> 5. **Cost Optimization** — avoid unnecessary costs
> 6. **Sustainability** — minimize environmental impact

**Q81: How do you design a highly available architecture on AWS?**

> - Deploy across **multiple Availability Zones** (min 2 AZs)
> - Use **Auto Scaling Groups** for compute
> - Use **Application Load Balancer** to distribute traffic
> - Use **RDS Multi-AZ** for database failover
> - Use **S3** for durable storage
> - Use **Route 53** health checks for DNS failover
> - Use **ElastiCache** to reduce database load
> - Design for **stateless** application tier

**Q82: What is the difference between EBS, EFS, and S3?**

> | EBS | EFS | S3 |
> |-----|-----|-----|
> | Block storage | File storage | Object storage |
> | Attached to one EC2 | Shared across EC2s | Accessed via API |
> | Like a hard drive | Like a network drive | Like cloud storage |
> | Low latency | NFS protocol | HTTP access |
> | Use: databases, OS | Use: shared files | Use: backups, static content |

---

# 8. KUBERNETES

## Basic

**Q83: What is Kubernetes?**

> Kubernetes (K8s) is an open-source container orchestration platform. It automates deployment, scaling, and management of containerized applications. It manages where containers run, restarts failed containers, scales based on load, and handles networking between services.

**Q84: What is a Pod?**

> A Pod is the smallest deployable unit in Kubernetes. It represents one or more containers that:
> - Share the same network namespace (same IP address)
> - Share storage volumes
> - Are scheduled together on the same node
>
> Usually one container per pod. Multi-container pods are for sidecars (logging, proxying).

**Q85: What is a Deployment?**

> A Deployment manages a set of identical pods. It provides:
> - **Desired state** — "I want 3 replicas of my app"
> - **Rolling updates** — update pods without downtime
> - **Rollback** — revert to a previous version
> - **Self-healing** — replaces crashed pods automatically

**Q86: What are the types of Kubernetes Services?**

> | Type | Description | Use Case |
> |------|-------------|----------|
> | **ClusterIP** | Internal only (default) | Service-to-service inside cluster |
> | **NodePort** | Exposes on each node's IP:port | Development, testing |
> | **LoadBalancer** | Provisions cloud load balancer | Production external access |
> | **ExternalName** | DNS alias to external service | Map to external database |

**Q87: What is a Namespace?**

> Namespaces provide virtual cluster separation within a single physical cluster. Used to:
> - Separate environments (dev, staging, production)
> - Separate teams
> - Apply resource quotas and network policies per namespace
>
> Default namespaces: `default`, `kube-system`, `kube-public`

## Intermediate

**Q88: What is the difference between a Deployment and a StatefulSet?**

> | Deployment | StatefulSet |
> |-----------|-------------|
> | Stateless applications | Stateful applications |
> | Pods are interchangeable | Each pod has a unique identity |
> | Random pod names (web-abc123) | Ordered names (db-0, db-1, db-2) |
> | Shared storage | Each pod gets its own persistent volume |
> | Use for: APIs, web servers | Use for: databases, Kafka, Elasticsearch |

**Q89: What is an Ingress?**

> Ingress manages external HTTP/HTTPS access to services in the cluster. It provides:
> - **Path-based routing** — `/api` → api-service, `/` → frontend-service
> - **Host-based routing** — `app.example.com` → app, `api.example.com` → api
> - **SSL/TLS termination** — HTTPS at the edge
> - **Load balancing** across service pods
>
> Requires an Ingress Controller (Nginx, Traefik, ALB).

**Q90: What are ConfigMaps and Secrets?**

> | ConfigMap | Secret |
> |-----------|--------|
> | Non-sensitive configuration | Sensitive data (passwords, keys) |
> | Plain text | Base64 encoded (not encrypted by default) |
> | Environment variables, config files | Credentials, certificates |
>
> Both can be mounted as environment variables or files in pods.

**Q91: What is a Helm Chart?**

> Helm is a package manager for Kubernetes. A Helm Chart is a collection of templates and values that deploy a complete application:
> - `Chart.yaml` — metadata
> - `values.yaml` — configurable defaults
> - `templates/` — Kubernetes manifests with template variables
>
> Benefits: reusable, versioned, configurable, can be shared via repositories.

**Q92: What are Liveness and Readiness probes?**

> | Liveness Probe | Readiness Probe |
> |---------------|-----------------|
> | Is the app alive? | Is the app ready to serve traffic? |
> | If fails: Kubernetes RESTARTS the pod | If fails: Kubernetes REMOVES pod from service |
> | Catches: app hangs, deadlocks | Catches: app still starting, loading cache |
>
> ```yaml
> livenessProbe:
>   httpGet:
>     path: /healthz
>     port: 8080
>   initialDelaySeconds: 15
>   periodSeconds: 10
> ```

## Advanced

**Q93: How does Kubernetes networking work?**

> Three networking rules in Kubernetes:
> 1. **Pod-to-Pod:** Every pod can communicate with every other pod without NAT
> 2. **Node-to-Pod:** Nodes can communicate with all pods
> 3. **Pod-to-Service:** Services provide stable endpoints (ClusterIP → pods)
>
> Implemented by CNI plugins (Calico, Flannel, Cilium, WeaveNet).
> Each pod gets its own IP. Services use kube-proxy for load balancing.

**Q94: What is RBAC in Kubernetes?**

> RBAC (Role-Based Access Control) manages who can do what:
> - **Role** — set of permissions within a namespace
> - **ClusterRole** — set of permissions cluster-wide
> - **RoleBinding** — binds a role to a user/group in a namespace
> - **ClusterRoleBinding** — binds a ClusterRole to a user/group cluster-wide
>
> Example: "dev-team can create/list pods in staging namespace but cannot delete anything in production."

**Q95: What is a Network Policy?**

> Network Policies control traffic flow between pods (like a firewall):
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: allow-frontend-only
> spec:
>   podSelector:
>     matchLabels:
>       app: api
>   ingress:
>     - from:
>         - podSelector:
>             matchLabels:
>               app: frontend
>       ports:
>         - port: 3000
> ```
> This allows only pods labeled `frontend` to access `api` pods on port 3000.

**Q96: What is a DaemonSet?**

> A DaemonSet ensures one pod runs on every node (or selected nodes). Use cases:
> - Log collectors (Fluentd, Filebeat)
> - Monitoring agents (Node Exporter, Datadog)
> - Network plugins (Calico, Cilium)
> - Storage daemons
>
> When a new node is added, the DaemonSet pod is automatically scheduled.

**Q97: How does Horizontal Pod Autoscaler (HPA) work?**

> HPA automatically scales pods based on metrics:
> 1. Metrics Server collects CPU/memory from pods
> 2. HPA controller checks metrics every 15 seconds
> 3. Calculates desired replicas: `desired = current * (currentMetric / targetMetric)`
> 4. Scales up/down within min/max bounds
> 5. Cooldown period prevents flapping
>
> Can scale on CPU, memory, or custom metrics (requests per second).

**Q98: Explain Kubernetes pod scheduling.**

> The Scheduler decides which node runs a pod:
> 1. **Filtering** — exclude nodes that don't meet requirements (resources, node selectors, taints)
> 2. **Scoring** — rank remaining nodes by criteria (least resource usage, pod spread)
> 3. **Binding** — assign pod to the highest-scored node
>
> You can influence scheduling with:
> - `nodeSelector` — simple label matching
> - `affinity/anti-affinity` — advanced rules (prefer, require)
> - `taints and tolerations` — prevent/allow scheduling on specific nodes
> - `topologySpreadConstraints` — distribute pods across zones

---

# 9. MONITORING & LOGGING

## Basic

**Q99: What is the difference between monitoring and observability?**

> | Monitoring | Observability |
> |-----------|---------------|
> | Tells you WHEN something is wrong | Tells you WHY something is wrong |
> | Pre-defined dashboards and alerts | Ability to ask new questions |
> | Metrics and thresholds | Metrics + Logs + Traces (three pillars) |
> | Reactive | Proactive debugging |

**Q100: What is Prometheus?**

> Prometheus is an open-source monitoring system that:
> - **Scrapes** metrics from targets (pull model)
> - Stores metrics in a **time-series database**
> - Provides **PromQL** query language
> - Has built-in **alerting** via Alertmanager
> - Integrates with **Grafana** for visualization

**Q101: What is Grafana?**

> Grafana is an open-source visualization and dashboarding tool. It connects to data sources (Prometheus, Elasticsearch, CloudWatch) and creates interactive dashboards with graphs, gauges, and tables. It also supports alerting.

## Intermediate

**Q102: Write a PromQL query for CPU usage above 80%.**

> ```promql
> 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
> ```
> This calculates non-idle CPU percentage per instance and filters those above 80%.

**Q103: What is the ELK Stack?**

> - **Elasticsearch** — search and analytics engine, stores and indexes logs
> - **Logstash** — log collection and processing pipeline (parse, transform, enrich)
> - **Kibana** — visualization and dashboards for Elasticsearch data
>
> Alternative: **EFK** replaces Logstash with **Fluentd** (lighter, K8s-native).

**Q104: What are the four golden signals of monitoring?**

> (From Google's SRE book)
> 1. **Latency** — time to serve a request
> 2. **Traffic** — requests per second
> 3. **Errors** — rate of failed requests
> 4. **Saturation** — how full resources are (CPU, memory, disk)
>
> Monitor these four for every service.

## Advanced

**Q105: How do you implement distributed tracing?**

> Distributed tracing tracks a request across multiple microservices:
> 1. Assign a unique **trace ID** at the entry point
> 2. Each service creates a **span** with start/end times
> 3. Pass trace ID through request headers
> 4. Collect spans in a tracing backend (Jaeger, Zipkin, AWS X-Ray)
> 5. Visualize as a waterfall diagram showing where time was spent
>
> Requires instrumenting code with OpenTelemetry SDK or auto-instrumentation.

---

# 10. SECRETS MANAGEMENT

## Basic

**Q106: Why should you never hardcode secrets?**

> - Committed to Git → exposed to everyone with repo access
> - Docker images → anyone who pulls the image can extract them
> - Logs → secrets may appear in error messages
> - Shared/copy-pasted → no audit trail, no rotation
>
> **Rule:** Secrets must be injected at runtime, not baked into code or images.

**Q107: What are common types of secrets in DevOps?**

> - Database passwords
> - API keys and tokens
> - SSH private keys
> - TLS/SSL certificates
> - Docker registry credentials
> - Cloud provider credentials (AWS keys)
> - Encryption keys
> - OAuth client secrets

## Intermediate

**Q108: Compare different secret management approaches.**

> | Method | Security | Use Case |
> |--------|----------|----------|
> | Environment variables | Low-Medium | Simple apps, local dev |
> | .env files (not committed) | Low | Local development |
> | CI/CD secrets (GitHub/Jenkins) | Medium | Pipeline credentials |
> | Kubernetes Secrets | Medium | K8s applications |
> | HashiCorp Vault | High | Enterprise, dynamic secrets |
> | AWS Secrets Manager | High | AWS-native apps |
> | SOPS (encrypted files) | Medium | GitOps workflows |

**Q109: How do Kubernetes Secrets work? Are they secure?**

> Kubernetes Secrets store data as base64-encoded (NOT encrypted) in etcd:
> ```yaml
> apiVersion: v1
> kind: Secret
> metadata:
>   name: db-creds
> data:
>   password: cGFzc3dvcmQxMjM=   # base64, NOT encrypted
> ```
>
> **Not secure by default.** To secure them:
> - Enable **etcd encryption at rest**
> - Use **RBAC** to restrict who can read secrets
> - Use **external secret operators** (External Secrets, Sealed Secrets)
> - Mount as **files** not environment variables (env vars can leak in logs)

**Q110: What is HashiCorp Vault?**

> Vault is a secrets management tool that provides:
> - **Centralized secrets storage** with encryption
> - **Dynamic secrets** — generate temporary database credentials on demand
> - **Secret rotation** — automatically rotate passwords
> - **Audit logging** — who accessed what and when
> - **Access policies** — fine-grained access control
> - **Multiple auth methods** — LDAP, Kubernetes SA, AppRole, tokens

## Advanced

**Q111: What are dynamic secrets in Vault?**

> Dynamic secrets are generated on-demand and automatically expire:
> 1. App requests database credentials from Vault
> 2. Vault creates a temporary user in the database with limited permissions
> 3. Returns credentials with a TTL (e.g., 1 hour)
> 4. After TTL expires, Vault revokes the credentials
>
> **Benefit:** No long-lived credentials, automatic rotation, each app gets unique credentials.

**Q112: How do you handle secrets in a GitOps workflow?**

> Options:
> 1. **Sealed Secrets** — encrypt secrets in Git, only the cluster can decrypt
> 2. **External Secrets Operator** — sync from Vault/AWS SM to K8s Secrets
> 3. **SOPS** — encrypt specific fields in YAML with PGP/KMS
> 4. **Vault Agent Injector** — sidecar injects secrets at pod startup
>
> **Never** store plain secrets in Git, even in private repositories.

---

# 11. LINUX & NETWORKING

## Basic

**Q113: What is the difference between TCP and UDP?**

> | TCP | UDP |
> |-----|-----|
> | Connection-oriented | Connectionless |
> | Reliable (guaranteed delivery) | Unreliable (best-effort) |
> | Ordered packets | No ordering guarantee |
> | Slower (overhead) | Faster (no overhead) |
> | HTTP, SSH, FTP, SMTP | DNS, DHCP, streaming, gaming |

**Q114: What happens when you type a URL in a browser?**

> 1. Browser checks cache for DNS record
> 2. DNS resolution: domain → IP address
> 3. Browser establishes TCP connection (3-way handshake)
> 4. TLS handshake (if HTTPS)
> 5. Browser sends HTTP request
> 6. Server processes request
> 7. Server sends HTTP response
> 8. Browser renders the HTML/CSS/JS

**Q115: What is the difference between `chmod 755` and `chmod 644`?**

> - `755` = `rwxr-xr-x` — owner: full, group: read+execute, others: read+execute
>   - Used for: executable scripts, directories
> - `644` = `rw-r--r--` — owner: read+write, group: read, others: read
>   - Used for: regular files, config files

**Q116: What is the difference between a process and a thread?**

> | Process | Thread |
> |---------|--------|
> | Independent execution | Runs within a process |
> | Own memory space | Shares process memory |
> | Heavier to create | Lightweight |
> | Crash doesn't affect others | Crash can affect other threads |
> | Communication via IPC | Direct memory sharing |

## Intermediate

**Q117: How do you troubleshoot a server that is slow?**

> Systematic approach:
> 1. **CPU:** `top`, `htop` — check if any process is consuming high CPU
> 2. **Memory:** `free -h` — check if swap is being used (low RAM)
> 3. **Disk:** `df -h`, `iostat` — check disk space and I/O wait
> 4. **Network:** `netstat`, `ss` — check connections, bandwidth
> 5. **Logs:** `tail -f /var/log/syslog` — check for errors
> 6. **Processes:** `ps aux --sort=-%mem` — top memory consumers
> 7. **Load average:** `uptime` — should be < number of CPU cores

**Q118: What is the difference between hard link and soft link?**

> | Hard Link | Soft Link (Symlink) |
> |-----------|-------------------|
> | Points to same inode (data) | Points to the file path |
> | File persists if original deleted | Breaks if original deleted |
> | Cannot cross filesystems | Can cross filesystems |
> | Cannot link to directories | Can link to directories |
> | `ln file hardlink` | `ln -s file symlink` |

**Q119: Explain the Linux boot process.**

> 1. **BIOS/UEFI** — hardware initialization, finds bootloader
> 2. **Bootloader (GRUB)** — loads the kernel
> 3. **Kernel** — initializes hardware, mounts root filesystem
> 4. **init/systemd** — first process (PID 1), starts services
> 5. **Runlevel/Target** — starts services for the configured target
> 6. **Login** — presents login prompt

## Advanced

**Q120: What is a reverse proxy vs a forward proxy?**

> | Forward Proxy | Reverse Proxy |
> |--------------|---------------|
> | Sits in front of clients | Sits in front of servers |
> | Client → Proxy → Internet | Internet → Proxy → Server |
> | Hides client identity | Hides server identity |
> | Use: content filtering, privacy | Use: load balancing, SSL, caching |
> | Example: Squid | Example: Nginx, HAProxy |

**Q121: Explain DNS resolution in detail.**

> 1. Browser checks its **cache**
> 2. Checks OS **hosts file** (`/etc/hosts`)
> 3. Queries **local DNS resolver** (e.g., router or ISP)
> 4. Resolver queries **Root DNS server** → returns TLD server
> 5. Queries **TLD server** (.com) → returns authoritative server
> 6. Queries **Authoritative DNS server** → returns IP address
> 7. Resolver caches the result (TTL)
> 8. Returns IP to the browser

---

# 12. DEVOPS CULTURE & PRACTICES

## Basic

**Q122: What is Infrastructure as Code (IaC)?**

> Managing infrastructure through code files instead of manual processes:
> - Version controlled (Git)
> - Reviewable (Pull Requests)
> - Repeatable (same code = same infrastructure)
> - Testable
> - Self-documenting
>
> Tools: Terraform, CloudFormation, Pulumi, Ansible

**Q123: What is the difference between mutable and immutable infrastructure?**

> | Mutable | Immutable |
> |---------|-----------|
> | Servers are updated in place | Servers are never modified after creation |
> | SSH in, install updates | Replace entire server with new version |
> | Configuration drift over time | Every server is identical |
> | Example: `apt upgrade` on running server | Example: deploy new Docker image |

## Intermediate

**Q124: What is SRE and how does it relate to DevOps?**

> SRE (Site Reliability Engineering) is Google's implementation of DevOps. Key concepts:
> - **SLIs** — measurable indicators of service quality
> - **SLOs** — reliability targets (e.g., 99.9% uptime)
> - **Error budgets** — how much downtime is acceptable
> - **Toil reduction** — automate repetitive manual work
>
> DevOps is a culture; SRE is a prescriptive way to implement it.

**Q125: What are SLI, SLO, and SLA?**

> | Term | Definition | Example |
> |------|-----------|---------|
> | **SLI** (Indicator) | Metric measuring service quality | Request latency < 200ms |
> | **SLO** (Objective) | Target value for an SLI | 99.9% of requests < 200ms |
> | **SLA** (Agreement) | Contract with consequences | 99.9% uptime or customer gets credit |
>
> SLI is what you measure. SLO is your internal target. SLA is the external promise.

**Q126: What is GitOps?**

> GitOps uses Git as the single source of truth for infrastructure and application state:
> - All changes go through Git (Pull Requests)
> - An operator (ArgoCD, FluxCD) watches the Git repo
> - Automatically syncs cluster state to match Git
> - Drift detection: if someone manually changes the cluster, it reverts
>
> Benefits: audit trail, easy rollback (revert commit), declarative.

## Advanced

**Q127: What is chaos engineering?**

> Chaos engineering proactively injects failures to test system resilience:
> - **Principle:** "What happens if this service goes down?"
> - **Process:** Hypothesis → Experiment → Observe → Learn
> - **Examples:** Kill a pod, simulate network latency, fill disk, CPU stress
> - **Tools:** Litmus, Chaos Monkey, Gremlin
> - **Goal:** Find weaknesses BEFORE they cause production outages

**Q128: Explain the concept of "Shift Left."**

> "Shift Left" means moving activities earlier in the development lifecycle:
> - **Shift-Left Testing:** Write tests before/during development, not after
> - **Shift-Left Security:** Scan for vulnerabilities during coding, not after deployment
> - **Shift-Left Quality:** Code reviews and linting during development
>
> The earlier you catch an issue, the cheaper it is to fix.

---

# 13. SCENARIO-BASED QUESTIONS (Advanced)

**Q129: A production deployment failed at 2 AM and users are seeing 500 errors. Walk me through your response.**

> 1. **Acknowledge** — check monitoring alerts, confirm the issue
> 2. **Assess impact** — how many users affected? Is it partial or full outage?
> 3. **Communicate** — notify the team via on-call channel
> 4. **Rollback** — immediately deploy the previous working version
>    - K8s: `kubectl rollout undo deployment/app`
>    - ArgoCD: revert Git commit
> 5. **Verify** — confirm rollback is successful, error rate dropping
> 6. **Investigate** — check logs, metrics, what changed in the deployment
> 7. **Root cause** — identify why it failed (missing env var? config issue?)
> 8. **Fix** — fix the issue, add tests to prevent recurrence
> 9. **Postmortem** — document timeline, root cause, and action items

**Q130: Your Kubernetes pod is in CrashLoopBackOff. How do you debug it?**

> ```bash
> # 1. Check pod status and recent events
> kubectl describe pod <pod-name>
>
> # 2. Check application logs
> kubectl logs <pod-name>
> kubectl logs <pod-name> --previous   # Logs from crashed container
>
> # 3. Common causes:
> #    - Application error (check logs)
> #    - Missing config/secrets (ConfigMap, Secret not found)
> #    - Wrong command or entrypoint
> #    - Liveness probe failing
> #    - Out of memory (OOMKilled — check resources.limits)
> #    - Image pull error (wrong tag, auth issue)
>
> # 4. Debug interactively
> kubectl run debug --image=busybox --rm -it -- sh
>
> # 5. Check events
> kubectl get events --sort-by=.lastTimestamp
> ```

**Q131: Your CI/CD pipeline is taking 30 minutes. How do you optimize it?**

> 1. **Parallelize** — run tests, lint, and scans concurrently
> 2. **Cache dependencies** — cache node_modules, pip packages
> 3. **Use faster runners** — upgrade CI machine, use self-hosted with SSD
> 4. **Docker layer caching** — reuse unchanged layers
> 5. **Only build what changed** — monorepo: detect changed services
> 6. **Skip unnecessary stages** — skip deployment on docs-only changes
> 7. **Slim test suite** — run unit tests in CI, integration tests on merge only
> 8. **Use pre-built base images** — don't install tools every run
> 9. **Incremental builds** — only compile changed code

**Q132: Your Terraform apply destroyed a production database. What happened and how do you prevent it?**

> **What happened:** Someone likely removed the resource from code, renamed it, or did `terraform destroy`.
>
> **Prevention:**
> 1. Use `lifecycle { prevent_destroy = true }` on critical resources
> 2. Enable **S3 versioning** on state bucket
> 3. Always run `terraform plan` and review before `apply`
> 4. Use **separate state files** per environment (never share prod/dev state)
> 5. Require **PR approval** for Terraform changes
> 6. Use **Sentinel/OPA policies** to block destructive operations
> 7. Enable **RDS deletion protection** and snapshots
> 8. Use `terraform plan -out=plan.tfplan` and `terraform apply plan.tfplan`

**Q133: How do you handle secret rotation without downtime?**

> 1. Update the secret in Vault/Secrets Manager with the new value
> 2. Application reads secrets dynamically (not from static env vars)
> 3. Old and new secrets both valid during transition period
> 4. Deploy new pods that pick up the new secret
> 5. Invalidate old secret after all pods are updated
>
> With Vault: dynamic secrets auto-rotate via TTL. With K8s: use Reloader or stakater to auto-restart pods when secrets change.

**Q134: You have 100 microservices. How do you manage their CI/CD pipelines?**

> 1. **Monorepo with path-based triggers** — detect which service changed, only build/test/deploy that service
> 2. **Shared pipeline templates** — reusable workflow (GitHub Actions reusable workflows, Jenkins shared libraries)
> 3. **GitOps with ArgoCD** — ApplicationSets auto-generate ArgoCD apps per service
> 4. **Helm charts** — one chart template with per-service values
> 5. **Convention over configuration** — standard directory structure, auto-detected by CI
> 6. **Service mesh** — Istio/Linkerd for traffic management, canary, observability

**Q135: A developer says "it works on my machine but not in the pipeline." How do you investigate?**

> 1. **Environment difference** — check Node/Python/Java version, OS
> 2. **Dependencies** — compare local `node_modules` vs CI fresh install
> 3. **Environment variables** — missing secrets/configs in CI
> 4. **File system** — case-sensitive (Linux CI) vs case-insensitive (Mac)
> 5. **Network** — CI may not have access to internal services
> 6. **Docker** — run the same Docker image locally as in CI
> 7. **Solution:** Containerize everything — same Docker image runs everywhere

---

# 14. RAPID-FIRE QUESTIONS (Quick Answer)

| # | Question | Answer |
|---|----------|--------|
| 136 | Default port for SSH? | 22 |
| 137 | Default port for HTTP / HTTPS? | 80 / 443 |
| 138 | What does `kubectl get pods -A` do? | Lists pods in ALL namespaces |
| 139 | What is etcd? | Key-value store holding all K8s cluster data |
| 140 | What format does Ansible use? | YAML |
| 141 | What language is Terraform written in? | Go; config in HCL |
| 142 | What is a Docker registry? | Storage for Docker images (e.g., Docker Hub, ECR) |
| 143 | What command shows Git commit history? | `git log` |
| 144 | What is port 6443 used for? | Kubernetes API server |
| 145 | What is an AMI? | Amazon Machine Image — template for EC2 instances |
| 146 | What does idempotent mean? | Same operation multiple times = same result |
| 147 | What is a sidecar container? | Helper container in the same pod (logging, proxy) |
| 148 | What is HPA in Kubernetes? | Horizontal Pod Autoscaler — auto-scales pods |
| 149 | What does `terraform fmt` do? | Formats Terraform code to standard style |
| 150 | What is a webhook? | HTTP callback — one system notifies another of an event |
| 151 | What is an init container? | Container that runs before app containers start |
| 152 | `git reset --hard` vs `git revert`? | Reset rewrites history (destructive); revert creates new commit |
| 153 | What is kube-proxy? | Network proxy on each node handling service routing |
| 154 | What is a Terraform workspace? | Isolated state for same config (e.g., dev/prod) |
| 155 | What does `docker system prune` do? | Removes unused containers, images, networks, volumes |

---

## Interview Tips for the Interviewer

1. **Start with basic questions** — let the candidate build confidence
2. **Ask follow-up questions** — "Can you explain that in more detail?"
3. **Scenario questions** reveal real experience — anyone can memorize definitions
4. **Look for problem-solving approach**, not just the right answer
5. **Red flags:**
   - Cannot explain the difference between Git merge and rebase
   - Has never used `kubectl logs` or `kubectl describe` to debug
   - Cannot explain how to handle secrets (says "put in .env and commit")
   - Cannot draw a basic CI/CD pipeline
6. **Green flags:**
   - Talks about automation and repeatability
   - Mentions security early (shift-left)
   - Can explain trade-offs (not just "use tool X")
   - Has opinions on deployment strategies
