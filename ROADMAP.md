# DevOps Mastery Roadmap 2026–27
> Mentor: Claude (Anthropic) | Student: jyotishiipsita
> Goal: Master AI-era DevOps and become a top-rated DevOps engineer by 2027
> Started: 2026-03-15

---

## The Mission
Build production-grade expertise across the full modern DevOps stack,
with hands-on projects at every phase. Every concept learned is immediately
applied to a real evolving project. Every session is documented for
revision, portfolio, and interview preparation.

---

## Learning Pillars

| Pillar | Topics | Status |
|--------|---------|--------|
| 1 | Foundation — Git advanced, Linux networking, Cloud basics | 🔄 In Progress |
| 2 | Containers — Docker, Kubernetes, Helm, EKS/GKE | ⏳ Pending |
| 3 | CI/CD + GitOps — GitHub Actions, ArgoCD, Flux, Terraform | ⏳ Pending |
| 4 | Observability — Prometheus, Grafana, OpenTelemetry, Loki | ⏳ Pending |
| 5 | DevSecOps — Vault, OPA, Trivy, SBOM, SLSA | ⏳ Pending |
| 6 | AI-Powered DevOps — AI agents, AIOps, Backstage IDP | ⏳ Pending |

---

## Phase 1 — Foundation Reinforcement
**Duration:** Months 1–2 | **Time:** ~60 hrs total
**Cloud:** AWS (Mumbai ap-south-1) + GCP

### Week 1 — Git Internals + Linux Networking

#### Git (COMPLETED ✅)
- [x] Session setup — SSH keys, GitHub repo, folder structure
- [x] Git object model — blob, tree, commit, tag
- [x] Branch internals — HEAD and ref pointers
- [x] Parent chain — how history is built
- [x] Rebase — standard and interactive (squash)
- [x] reflog — recovery from any mistake
- [x] Pre-commit hooks
- [x] Cherry-pick + conflict resolution
- [x] Notes, lab commands, interview Q&A documented

#### Linux Networking (NEXT ⏳)
- [ ] `ss`, `ip`, `netstat` — active connections and ports
- [ ] `dig`, `nslookup` — DNS resolution deep dive
- [ ] `traceroute`, `mtr` — network path analysis
- [ ] `iptables` — firewall rules
- [ ] `curl -v` — full HTTP/TLS handshake visibility
- [ ] Network namespaces — foundation of Docker networking
- [ ] `/etc/hosts`, `/etc/resolv.conf` — DNS config
- [ ] VPC networking concepts on AWS

#### Cloud CLI (PENDING ⏳)
- [ ] AWS CLI — EC2, VPC, S3, IAM via CLI only
- [ ] GCP CLI — Compute Engine, Cloud Storage via gcloud
- [ ] Launch EC2 via CLI, SSH, install nginx
- [ ] Create VPC with public/private subnets

### Week 2 — Cloud Deep Dive (PENDING ⏳)
- [ ] AWS VPC, subnets, security groups in depth
- [ ] IAM roles, policies, best practices
- [ ] S3 lifecycle, versioning, policies
- [ ] Route53 basics
- [ ] GCP equivalents — VPC, IAM, GCS

---

## Phase 2 — Containers and Kubernetes
**Duration:** Months 3–4 | **Time:** ~70 hrs total

### Topics
- [ ] Docker internals — images, layers, networking, volumes
- [ ] Docker Compose — multi-service applications
- [ ] Kubernetes core — pods, deployments, services, ingress
- [ ] RBAC — roles, bindings, service accounts
- [ ] Helm — charts, values, upgrades, rollbacks
- [ ] EKS — managed Kubernetes on AWS
- [ ] GKE — managed Kubernetes on GCP

### Project milestone
Deploy containerized microservices app to EKS with Helm

---

## Phase 3 — CI/CD + GitOps + IaC
**Duration:** Months 5–7 | **Time:** ~100 hrs total

### Topics
- [ ] GitHub Actions — workflows, secrets, reusable actions
- [ ] ArgoCD — GitOps, app of apps pattern
- [ ] Flux CD — alternative GitOps operator
- [ ] Terraform — AWS + GCP IaC
- [ ] Pulumi — IaC with real programming languages
- [ ] OpenTofu — open source Terraform
- [ ] Atlantis — Terraform pull request automation
- [ ] Python scripting for DevOps (enters here)

### Project milestone
Full GitOps pipeline — push to Git = deploy to Kubernetes automatically

---

## Phase 4 — Observability + DevSecOps
**Duration:** Months 8–10 | **Time:** ~100 hrs total

### Topics
- [ ] Prometheus — metrics collection and alerting
- [ ] Grafana — dashboards and visualization
- [ ] OpenTelemetry — unified observability standard
- [ ] Loki — log aggregation
- [ ] Jaeger — distributed tracing
- [ ] HashiCorp Vault — secrets management
- [ ] OPA / Kyverno — policy as code
- [ ] Trivy / Snyk — vulnerability scanning
- [ ] SBOM + SLSA — supply chain security

### Project milestone
Full observability stack + automated security gates in CI/CD

---

## Phase 5 — AI-Powered DevOps + Platform Engineering
**Duration:** Months 11–14 | **Time:** ~110 hrs total

### Topics
- [ ] AI agents in CI/CD pipelines
- [ ] AIOps — self-healing infrastructure
- [ ] LLM APIs in automation scripts (Python)
- [ ] Backstage — Internal Developer Platform
- [ ] Platform engineering principles
- [ ] FinOps — cost optimization

### Project milestone
Production-grade app with AI-powered operations + full IDP

---

## The Hands-On Project (Evolves Every Phase)
> One real-world microservices application that grows with every phase

| Phase | What gets added |
|-------|----------------|
| 1 | App deployed on EC2, managed via CLI |
| 2 | Containerized, deployed to EKS with Helm |
| 3 | Full GitOps pipeline, IaC for all infra |
| 4 | Observability stack, security scanning |
| 5 | AI agents, self-healing, IDP integration |

**Final deliverable:** A portfolio-grade project showing the complete
modern DevOps stack — visible to every future employer at
github.com/jyotishiipsita/devops-mentorship

---

## Repository Structure
```
devops-mentorship/
├── ROADMAP.md                          ← this file — north star
├── README.md
├── week-01-git-linux/
│   ├── labs/
│   │   └── git-object-lab/            ← git internals lab
│   ├── scripts/                       ← bash scripts
│   └── notes.md                       ← quick session log
├── notes/
│   ├── git-session/
│   │   ├── git-notes.md               ← concepts and theory
│   │   ├── git-lab.md                 ← commands and outputs
│   │   └── git-interview-qns-ans.md   ← interview preparation
│   ├── linux-session/
│   ├── docker-session/
│   ├── kubernetes-session/
│   ├── cicd-session/
│   ├── iac-session/
│   ├── observability-session/
│   ├── devsecops-session/
│   └── ai-devops-session/
└── projects/                          ← evolving hands-on project
```

---

## Rules We Follow
1. Learn by doing — every concept has a hands-on lab
2. Update notes after every session
3. Push to GitHub at the end of every session
4. Move to next week only when current week checklist is done
5. Interview prep docs updated with every new topic
6. One CONTINUATION-PROMPT.md always kept up to date

---

## End State (Target: End of 2027)
- AI-era DevOps engineer with production-ready portfolio
- Capable of designing and operating full cloud-native platforms
- Skilled in AI-augmented operations and platform engineering
- Top 5% skill set in the DevOps market
