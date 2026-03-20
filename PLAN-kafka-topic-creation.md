# Plan: Auto-create Kafka Topics After Provisioning

*Created: 2026-03-20*

## Goal

Automatically create Kafka topics (defined in `terraform.tfvars`) after the Droplet is provisioned, so every `terraform apply` yields a ready-to-use broker with all required topics — and new topics can be added later without destroying the Droplet.

## Approach: `terraform_data` with `remote-exec` provisioner

Add a `terraform_data` resource that depends on the Droplet being fully provisioned. This keeps topic creation decoupled from the Docker install step and makes it easy to add or change topics later.

### Why `terraform_data` over `null_resource`?

- Built-in to Terraform since 1.4 — no extra provider required (we already require `>= 1.5`).
- `null_resource` + `hashicorp/null` is legacy; `terraform_data` is the recommended replacement.
- Uses `triggers_replace` which accepts any type (no need to `jsonencode`).

### Why not the Kafka Terraform provider?

- Adds a provider dependency and requires the broker to be network-reachable from the machine running Terraform at plan time.
- Overkill for a single-broker ephemeral dev setup.

### Why a separate resource instead of adding to the existing `remote-exec`?

- Keeps concerns separated (infra setup vs. application-level seeding).
- Can be extended independently (e.g., add more topics, change partitions) without re-triggering Docker install.
- Re-runs only when the topic list or Droplet changes — no need to destroy the Droplet to add topics.

## Changes

### 1. `variables.tf` — add `kafka_topics` variable

```hcl
variable "kafka_topics" {
  description = "Kafka topics to create after provisioning"
  type = list(object({
    name               = string
    partitions         = optional(number, 1)
    replication_factor = optional(number, 1)
  }))
}
```

No default is set — all topics must be defined in `terraform.tfvars`. This makes `terraform.tfvars` the single source of truth for which topics exist.

### 2. `terraform.tfvars` — define topics

```hcl
kafka_topics = [
  { name = "esb.portal.user.consume.v1" },
]
```

### 3. `main.tf` — add `terraform_data.kafka_topics`

```hcl
resource "terraform_data" "kafka_topics" {
  depends_on = [digitalocean_droplet.main]

  triggers_replace = [var.kafka_topics, digitalocean_droplet.main.id]

  connection {
    type        = "ssh"
    user        = "root"
    private_key = tls_private_key.droplet.private_key_openssh
    host        = digitalocean_droplet.main.ipv4_address
  }

  provisioner "remote-exec" {
    inline = concat(
      # Wait for Kafka broker to be ready (up to 60s)
      [
        "for i in $(seq 1 12); do docker exec ${var.project_name}-kafka-cloud kafka-broker-api-versions --bootstrap-server localhost:9092 > /dev/null 2>&1 && break || sleep 5; done",
      ],
      # Create each topic
      [for t in var.kafka_topics :
        "docker exec ${var.project_name}-kafka-cloud kafka-topics --create --topic ${t.name} --partitions ${t.partitions} --replication-factor ${t.replication_factor} --bootstrap-server localhost:9092 --if-not-exists"
      ]
    )
  }
}
```

Key details:
- **Readiness check**: Polls `kafka-broker-api-versions` every 5s (up to 60s) before attempting topic creation. More reliable than a fixed `sleep`.
- **`--if-not-exists`**: Makes topic creation idempotent — safe to re-run.
- **`triggers_replace`**: Re-runs when the topic list changes or the Droplet is recreated.

## Files touched

| File | Change |
|------|--------|
| `variables.tf` | Add `kafka_topics` variable (no default) |
| `terraform.tfvars` | Define the list of Kafka topics |
| `main.tf` | Add `terraform_data.kafka_topics` resource |

## Adding a new topic later

No `terraform destroy` needed. Just:

1. Add the new topic to the `kafka_topics` list in `terraform.tfvars`.
2. Run `terraform apply`.

This works because `terraform_data.kafka_topics` is a **separate resource** from the Droplet. When you add a topic, the Droplet itself hasn't changed — Terraform has no reason to recreate it or re-run its provisioner. Instead, `triggers_replace` detects the change in `var.kafka_topics` and recreates only the `terraform_data.kafka_topics` resource, re-running the topic creation script via SSH against the existing Droplet.

The `--if-not-exists` flag ensures previously created topics are skipped — only the new topic gets created.

Without this separate resource, adding a topic would require either:
- Tainting/destroying the Droplet just to re-run provisioning (wasteful, causes downtime).
- SSH-ing in manually to create topics (defeats the purpose of IaC).

## Verification

After `terraform apply`:
```bash
ssh -i generated/id_ed25519 root@<droplet_ip> \
  "docker exec <project_name>-kafka-cloud kafka-topics --list --bootstrap-server localhost:9092"
```

Expected output should include all topics defined in `terraform.tfvars`.
