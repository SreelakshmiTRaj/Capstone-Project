# Automated Application Delivery on AWS with Jenkins, Terraform, Ansible & ECS
> 
An end-to-end DevOps pipeline that builds, scans, containerizes and deploys a React + Vite **Quiz App** to **Amazon ECS (Fargate)** using **Blue/Green deployment**, with infrastructure provisioned by **Terraform**, servers configured by **Ansible**, and monitoring/alerting through **CloudWatch + Grafana**.
> 
> 📄 **Full project documentation:** [`Capstone_Project.pdf`](./Capstone_Project.pdf)

## Architecture Diagram
![Architecture](images/architecture.png)

## Highlights

- **Infrastructure as Code**: VPC, subnets, NAT/IGW, IAM, ECR, ECS, ALB and EC2 provisioned by Terraform through a Jenkins infrastructure pipeline.
- **Configuration management**: Ansible installs Docker and deploys SonarQube and Grafana on a dedicated managed EC2.
- **Secure CI/CD**: lint, build, SonarQube quality gate and Trivy scan (fails on HIGH/CRITICAL) before anything reaches ECR.
- **Blue/Green releases**: new version goes to the Green ECS service, is health-checked, then the ALB listener is switched. Rollback is a listener flip back to Blue.
- **Observability**: CloudWatch logs/metrics, Grafana dashboards, and alert rules tested by deliberately triggering and clearing them.
- **No long-lived AWS keys**: Jenkins and Grafana use EC2 IAM roles.

## Tech Stack

| Area | Tools |
|---|---|
| Infrastructure | Terraform, Amazon VPC, IGW, NAT Gateway, IAM |
| Configuration | Ansible |
| CI/CD | Jenkins, Git / AWS CodeCommit |
| Containers | Docker (multi-stage, Nginx), Amazon ECR, Amazon ECS Fargate |
| Traffic | Application Load Balancer (Blue/Green target groups) |
| Security & quality | SonarQube, Trivy |
| Monitoring | Amazon CloudWatch, Grafana |

## Architecture Overview

- **Network**: one VPC with 2 public + 2 private subnets across two Availability Zones. The ALB and NAT Gateway sit in public subnets; ECS tasks run in private subnets.
- **Control EC2** (bootstrap): runs Jenkins, Terraform, Ansible, AWS CLI, Docker and Trivy. It is created manually because Jenkins must exist before it can run the Terraform pipeline.
- **Managed EC2** (Terraform-provisioned, Ansible-configured): hosts SonarQube (`:9000`) and Grafana (`:3000`) as Docker containers.
- **ECS**: Blue and Green services, each behind its own ALB target group. The ALB listener's default target decides which version is live.

## Pipelines

**1. Infrastructure pipeline** (`Jenkinsfile-infrastructure`)
`Checkout → terraform init → fmt/validate → plan → apply`

**2. Application pipeline** (`Jenkinsfile`)

```
Checkout → npm ci → Lint → Build → SonarQube analysis → Quality Gate
   → Docker build → Trivy scan → Push to ECR
   → Deploy to Green → Validate health → Shift traffic to Green
```

Images are tagged with the Jenkins build number, so the exact image that passed the gates is the one deployed.

## Repository Structure

```
.
├── app/ (or repo root)       # React + Vite Quiz App, Dockerfile
├── terraform/                # vpc, subnet, nat, route_table, security_group,
│                             # iam, ec2, ecr, ecs, alb, outputs, variables
├── ansible/                  # inventory.ini, ansible.cfg, site.yml,
│   └── roles/                # docker, sonarqube, grafana
├── ecs-task-definition.json
├── Jenkinsfile               # application CI/CD pipeline
├── Jenkinsfile-infrastructure
└── images/                   # README screenshots
```

> Adjust the layout above to match the repo.

## Getting Started

**Prerequisites**: AWS account with sufficient permissions; a Control EC2 with Jenkins, Terraform, AWS CLI, Ansible (+ Docker collection), Docker and Trivy installed; a key pair for Control → Managed EC2 SSH; the app in a Git/CodeCommit repo.

1. **Containerize the app**: `docker build -t quiz-app .` and run locally to verify.
2. **Provision infrastructure**: run the Jenkins infrastructure pipeline (or `terraform init && terraform plan && terraform apply` from `terraform/`).
3. **Configure the managed node**: set its public IP in `inventory.ini`, then:
   ```bash
   ansible managed -m ping
   ansible-playbook site.yml
   ```
4. **Wire up Jenkins**: add the SonarQube token as a *Secret text* credential, configure the SonarQube server and the `/sonarqube-webhook/` webhook.
5. **Run the application pipeline**, then open the ALB DNS name.
6. **Add Grafana**: add CloudWatch as a data source (IAM role auth), build the dashboard and alert rules.

Remember to replace placeholders such as `<AWS_ACCOUNT_ID>`, `<AWS_REGION>`, `<IMAGE_TAG>` and `<MANAGED_EC2_PUBLIC_IP>`.

## Results

### Application served through the ALB
![App via ALB](images/app-via-alb.png)

### DevOps services deployed by Ansible
| SonarQube | Grafana |
|---|---|
| ![SonarQube](images/sonarqube.png) | ![Grafana](images/grafana.png) |

### Security gate and image publishing
Trivy reports zero HIGH/CRITICAL vulnerabilities, and the validated image lands in ECR.

![Trivy scan](images/trivy-scan.png)
![ECR image](images/ecr-image.png)

### Blue/Green traffic shift and rollback
Jenkins repoints the ALB listener to Green; a manual approval step reverses it to Blue.

![Shift to Green](images/shift-to-green.png)
![Rollback](images/rollback.png)

### Monitoring dashboards
| ECS CPU Utilization | ALB Request Count |
|---|---|
| ![ECS CPU](images/dashboard-ecs-cpu.png) | ![ALB requests](images/dashboard-alb-requests.png) |

### Alert testing

**High ECS CPU (> 80%)**: a CPU stress test on the running task fires the alert; it returns to normal once the load stops.

| Firing | Recovered |
|---|---|
| ![CPU firing](images/alert-cpu-firing.png) | ![CPU recovered](images/alert-cpu-recovered.png) |

**Unhealthy ALB targets (> 0)**: an ECS target is made unavailable, then restored.

| Firing | Normal |
|---|---|
| ![Unhealthy firing](images/alert-unhealthy-firing.png) | ![Unhealthy normal](images/alert-unhealthy-normal.png) |

## Notes & Possible Improvements

- Control EC2 is a deliberate bootstrap exception; everything else is Terraform-managed.
- Blue/Green is implemented with two ECS services plus ALB listener switching, not CodeDeploy.
- Ideas: HTTPS on the ALB, remote Terraform state (S3 + DynamoDB), automated rollback on failed validation, notification channel (SNS/Slack) for Grafana alerts.

## Author

**Sreelakshmi T Raj**: [Quiz App source](https://github.com/SreelakshmiTRaj/Quiz-App)
