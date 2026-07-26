---
name: bohrium-project
description: "Manage Bohrium projects via bohr CLI or open.bohrium.com API. Use when: user asks about creating/listing/deleting projects on Bohrium, managing project members, or setting cost limits. NOT for: job submission, node management, or image management."
---

# SKILL: Bohrium Project Management

## Overview

Manage projects on the Bohrium platform. **Prefer `bohr` CLI**; fall back to the API only for operations the CLI doesn't support (member management, cost limits, renaming).

Projects are the organizational containers for Nodes, Jobs, Images, and Datasets — and the basic unit for team collaboration and cost management.

## Authentication

```json
"bohrium-project": {
  "enabled": true,
  "apiKey": "YOUR_BOHR_ACCESS_KEY",
  "env": {
    "BOHR_ACCESS_KEY": "YOUR_BOHR_ACCESS_KEY"
  }
}
```

Only configure `BOHR_ACCESS_KEY` for this skill. Helper scripts handle any legacy CLI compatibility internally.

When calling the `bohr` CLI directly, map `BOHR_ACCESS_KEY` to the legacy variable that the CLI reads:

```bash
export ACCESS_KEY="$BOHR_ACCESS_KEY"
```

## Prerequisites: Install bohr CLI

```bash
# macOS
/bin/bash -c "$(curl -fsSL https://dp-public.oss-cn-beijing.aliyuncs.com/bohrctl/1.0.0/install_bohr_mac_curl.sh)"

# Linux
/bin/bash -c "$(curl -fsSL https://dp-public.oss-cn-beijing.aliyuncs.com/bohrctl/1.0.0/install_bohr_linux_curl.sh)"

source ~/.bashrc  # or source ~/.zshrc
export PATH="$HOME/.bohrium:$PATH"
export ACCESS_KEY="$BOHR_ACCESS_KEY"
```

---

## List Projects

```bash
bohr project list               # Table format
bohr project list --json        # JSON format
bohr project list --csv         # CSV format
```

**JSON fields:**

| Field | Description |
|-------|-------------|
| `projectId` | Project ID |
| `name` | Project name |

---

## Create Project

```bash
bohr project create -n "my-experiment"
bohr project create -n "my-experiment" -m 5000          # Monthly cost limit
bohr project create -n "my-experiment" -t 10000         # Total cost limit
```

**Parameters:**

| Parameter | Short | Required | Description |
|-----------|-------|----------|-------------|
| `--name` | `-n` | Yes | Project name (default "default") |
| `--month_cost_limit` | `-m` | No | Monthly cost limit |
| `--total_cost_limit` | `-t` | No | Total cost limit |

---

## Delete Project

```bash
bohr project delete YOUR_PROJECT_ID
```

> **Warning**: Deleting a project is **irreversible** — all jobs and images under the project will be removed and cannot be recovered.

---

## Roles & Permissions

Bohrium projects have 3 roles:

| Role | Description |
|------|-------------|
| Creator | The user who created the project; exactly one per project, non-transferable |
| Admin | Appointed by the creator; can have multiple; can be revoked at any time |
| Member | Users added to the project; default role |

### Permission Matrix

| Module | Permission | Creator | Admin | Member |
|--------|-----------|:-------:|:-----:|:------:|
| Project | Rename project | ✓ | ✓ | ✗ |
| Project | Delete project | ✓ | ✗ | ✗ |
| Members | Add/remove members | ✓ | ✓ | ✗ |
| Members | Promote/demote admins | ✓ | ✗ | ✗ |
| Budget | View/adjust project & member budgets | ✓ | ✓ | ✗ |
| Nodes | View/manage all project nodes | ✓ | ✓ | ✗ |
| Jobs | View/manage all project jobs | ✓ | ✓ | ✗ |
| Images | View/manage all project images | ✓ | ✓ | ✗ |
| Billing | View/download billing reports | ✓ | ✓ | ✗ |

> **Important**: Costs incurred by members are charged directly to the project creator's wallet balance.

---

## Budget Management

### Project Budget

Creators and admins can set the project's total budget (optional). If not set, the default is "unlimited".

When the project's total cost exceeds the budget, members cannot submit new jobs or start new nodes.

### Member Budget

Individual spending limits can be assigned per member:
- "Even split": Divide the project budget equally among all members
- "Uniform": Set the same limit for each member
- Manual: Set different limits for different members

Set project cost limit via API:
```python
requests.post(f"{BASE}/set_cost_limit", headers=HEADERS_JSON,
    json={"projectId": YOUR_PROJECT_ID, "costLimit": 5000})
```

---

## Shared Resources

### Shared Disk (/share)

Each project has 1TB of free shared storage with read/write access for all members.

- Access the `/share` directory via Web Shell or the file management page
- Data persists after node release
- Additional capacity can be purchased

### Shared Images

All project members can see custom images created by other members in the Bohrium Image Center, making it easy to share development environments.

---

## Billing

| Item | Description |
|------|-------------|
| Compute resources | Billed by duration of resource usage; prices vary by configuration |
| Dev nodes | Billed continuously while running; stop or delete when not in use |
| Personal storage (/personal) | 500GB free; additional capacity requires purchase |
| Project storage (/share) | 1TB free; additional capacity requires purchase |

- Account balance is deducted every 5 minutes
- Warning email sent when balance drops below threshold (default ¥100)
- Cannot submit new jobs when balance reaches zero

---

## Quotas

| Resource | Limit |
|----------|-------|
| Projects | 4 per user (only self-created; joined projects don't count) |
| Nodes | 4 per user per project |
| Concurrent running jobs | 100 per user |
| Custom images | 10 per project |
| Project shared disk | 1TB per project |
| Personal data disk | 500GB per user per project |

> Contact Bohrium support to increase quotas.

---

## API Supplement (CLI Unsupported)

The following operations are not covered by the bohr CLI and require the API:

```python
import os, requests

AK = os.environ.get("BOHR_ACCESS_KEY", "")
BASE = "https://open.bohrium.com/openapi/v2/project"
HEADERS = {"Authorization": f"Bearer {AK}"}
HEADERS_JSON = {**HEADERS, "Content-Type": "application/json"}

# Detailed project list (with cost, member count, etc.)
r = requests.get(f"{BASE}/list", headers=HEADERS)
# Returns: {items: [{id, name, totalCost, monthCost, userCount, projectRole, ...}]}

# Lightweight project list (id + name only)
r = requests.get(f"{BASE}/lite_list", headers=HEADERS)

# Rename project
requests.post(f"{BASE}/set_name", headers=HEADERS_JSON,
    json={"projectId": YOUR_PROJECT_ID, "name": "new-name"})

# Set cost limit
requests.post(f"{BASE}/set_cost_limit", headers=HEADERS_JSON,
    json={"projectId": YOUR_PROJECT_ID, "costLimit": 5000})

# View project members
r = requests.get(f"{BASE}/YOUR_PROJECT_ID/users", headers=HEADERS)
# Returns: {items: [{userId, userName, email, projectRole, cost, ...}]}

# Add member (by email)
requests.post(f"{BASE}/add_user", headers=HEADERS_JSON,
    json={"projectId": YOUR_PROJECT_ID, "email": "user@example.com"})

# Remove member
requests.post(f"{BASE}/del_user", headers=HEADERS_JSON,
    json={"projectId": YOUR_PROJECT_ID, "userId": 12345})

# Promote/demote admin
requests.post(f"{BASE}/manager/add", headers=HEADERS_JSON,
    json={"projectId": YOUR_PROJECT_ID, "userId": 12345})
requests.post(f"{BASE}/manager/del", headers=HEADERS_JSON,
    json={"projectId": YOUR_PROJECT_ID, "userId": 12345})

# Recover deleted member
requests.put(f"{BASE}/YOUR_PROJECT_ID/recovery_user", headers=HEADERS_JSON,
    json={"userId": 12345})
```

### Project Role API Values

| projectRole | Meaning |
|-------------|---------|
| 1 | Creator / Admin |
| 3 | Regular member |

---

## Unavailable Endpoints

The following endpoints are **not accessible** via openapi auth (return 404):

| Endpoint | Reason |
|----------|--------|
| `POST /project/join` | Route forwarding path mismatch |
| `POST /project/share_status` | Same |
| `GET /project/available` | Registered in upstream AK v2 Group; unreachable via openapi gateway auth |

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| `AccessKey Invalid` | Direct `bohr` calls are missing the legacy variable name | Run `export ACCESS_KEY="$BOHR_ACCESS_KEY"` and retry |
| Can't find newly created project | New project is at end of list | `bohr project list --json` to see all |
| Failed to remove member | Wrong userId | Get userId via API `/{id}/users` first |
| Adding member has no effect | Email doesn't exist | Ensure target user is registered on Bohrium |
| Project count limit reached | Max 4 self-created projects per user | Delete unused projects or contact support |
| Member can't submit jobs | Project or member budget exceeded | Creator/admin adjusts budget |
| Insufficient balance | Account balance is zero | Top up to resume usage |
