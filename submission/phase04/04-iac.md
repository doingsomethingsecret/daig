# Phase 4 — Infrastructure as Code (IaC)

## Objective

Treat infrastructure as reviewable code and understand Terraform and Ansible idempotency.

---

## 1. Terraform Validation

Terraform configuration reviewed:

```text
infra/aws
```

Commands executed:

```bash
terraform init -backend=false
terraform validate
terraform fmt -check -recursive
```

Results:

- `terraform init -backend=false` — PASS
- `terraform validate` — PASS
- `terraform fmt -check -recursive` — PASS after formatting

The initial formatting check reported:

```text
data.tf
```

The formatting issue was corrected with:

```bash
terraform fmt -recursive
```

The formatting check was then executed again successfully with no output.

Evidence:

- Terraform validation and formatting output

---

## 2. Production Safety Controls

### Control 1 — ECS Lifecycle: `ignore_changes`

The ECS service contains:

```hcl
lifecycle {
  ignore_changes = [desired_count]
}
```

Application Auto Scaling owns the runtime ECS `desired_count`.

This prevents Terraform from resetting the ECS service's runtime task count when Application Auto Scaling changes it.

This is important during the iftar traffic spike. Application Auto Scaling can increase the number of ECS tasks to handle increased traffic, while Terraform does not immediately try to reduce the task count back to the Terraform configuration value.

Without this control, a later Terraform apply could reduce live capacity during a traffic spike.

Evidence:

- `29-ecs-lifecycle.png`

---

### Control 2 — Database TLS Enforcement

The PostgreSQL connection string contains:

```text
sslmode=require
```

This requires the client connection to use TLS when connecting to the RDS PostgreSQL database.

The configuration also documents server-side TLS enforcement using:

```text
rds.force_ssl=1
```

This provides protection against unencrypted database connections.

Evidence:

- `30-db-tls.png`

---

### Control 3 — VPC Flow Logs

The AWS network configuration contains:

```hcl
resource "aws_flow_log" "main" {
```

VPC Flow Logs provide network traffic records for the VPC.

They support:

- Network troubleshooting
- Security investigation
- Identification of unexpected traffic
- Network auditing

Evidence:

- `31-vpc-flow-logs.png`

---

## 3. Remote Terraform State

The AWS Terraform configuration contains a commented S3 backend:

```hcl
# backend "s3" {
```

The remote backend is commented out because this exercise does not require a real remote-state setup or AWS credentials.

Terraform therefore uses local state for this exercise.

### What breaks if two engineers apply against local state?

If two engineers work with separate local Terraform state files, they do not have one shared source of truth.

This can cause:

- State divergence
- Conflicting infrastructure changes
- Concurrent applies
- Engineers not seeing each other's changes
- Infrastructure drift

A remote backend such as S3 provides shared state, while state locking prevents multiple engineers from modifying the same state concurrently.

Evidence:

- `32-remote-state.png`

---

## 4. Ansible Idempotency

The Docker role is located at:

```text
ansible/roles/docker/tasks/main.yml
```

One important task is:

```yaml
- name: Ensure Docker is running
  ansible.builtin.service:
    name: docker
    state: started
    enabled: true
```

Ansible is designed to be idempotent.

The first execution may report tasks as:

```text
changed
```

because the desired configuration is being applied.

When the same playbook is executed again, resources that already match the desired state should report:

```text
ok
```

instead of being changed again.

The Docker service task uses:

```yaml
state: started
```

Ansible checks the current state and only makes a change if Docker is not already running.

Other configuration tasks also need to be idempotent. For example, the Docker daemon configuration should be managed declaratively so that running the playbook again does not unnecessarily modify the configuration or restart Docker.

Evidence:

- `33-ansible-idempotency.png`

---

## 5. Iftar Checkpoint

### What does `lifecycle { ignore_changes = [desired_count] }` prevent during the iftar spike?

It prevents Terraform from resetting the ECS service's runtime `desired_count` when Application Auto Scaling has already increased the number of running tasks.

Example:

```text
Normal traffic
    ↓
3 ECS tasks
    ↓
Iftar traffic spike
    ↓
Application Auto Scaling increases capacity
    ↓
10 ECS tasks
```

Without `ignore_changes`:

```text
Terraform apply
    ↓
Terraform sees runtime desired_count = 10
    ↓
Terraform configuration specifies a lower value
    ↓
Terraform attempts to reset desired_count
    ↓
Live capacity can be reduced during the traffic spike
```

With:

```hcl
ignore_changes = [desired_count]
```

Terraform leaves the runtime task count under the control of Application Auto Scaling.

Therefore:

```text
Terraform
    → manages infrastructure configuration

Application Auto Scaling
    → manages runtime desired_count
```

This prevents Terraform from fighting the autoscaling system and accidentally removing capacity during the iftar traffic spike.

---

## 6. Evidence Summary

| Evidence | Description |
|---|---|
| `29-ecs-lifecycle.png` | ECS lifecycle `ignore_changes = [desired_count]` |
| `30-db-tls.png` | PostgreSQL TLS configuration |
| `31-vpc-flow-logs.png` | AWS VPC Flow Logs resource |
| `32-remote-state.png` | Commented remote S3 backend |
| `33-ansible-idempotency.png` | Ansible Docker role and idempotent service configuration |

---

## Phase 4 Result

Terraform configuration was initialized and validated without applying infrastructure.

Formatting was corrected and verified.

Three production-safety controls were identified:

1. ECS lifecycle protection for `desired_count`
2. PostgreSQL TLS requirement
3. VPC Flow Logs

The reason for disabling the remote backend during the exercise and the risks of local Terraform state were documented.

Ansible idempotency was reviewed through the Docker role.

No cloud infrastructure was deployed or modified as part of this phase.
