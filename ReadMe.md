# Cloud-Native Security & Kubernetes Hardening Lab

## What this is

I built this as a small hands-on security lab on AWS to get more practical experience with cloud and Kubernetes security.

The setup is intentionally simple: one Ubuntu server running K3s, a small Nginx frontend and backend application, and a few security tools around it. I then tested different security controls, tried a few things that shouldn't be allowed, watched what the security tools detected, and documented the results.

This is a lab environment, not a production deployment. All the results below came from tests I ran myself, along with the screenshots and command outputs from the lab.

**Stack:** AWS EC2 (Ubuntu 24.04) → K3s → Nginx frontend + backend app  
**Namespace:** `security-lab`  
**Security tools:** RBAC, NetworkPolicy, Kubernetes Secrets, Trivy, Falco, CloudTrail

---

## Results at a glance

| Area | What I tested | Result |
|---|---|---|
| RBAC | Checked what two service accounts could and couldn't do | Analyst could get pods but not delete them. Developer could create deployments but not delete them |
| NetworkPolicy | Tried connecting to the backend from a random BusyBox pod | Worked before the policy; connection was refused after the policy |
| Secrets | Created a test Kubernetes Secret and viewed its value | Values were only base64-encoded, not encrypted |
| Trivy | Scanned both container images | `nginx` had 137 findings and `http-echo` had 72 |
| Falco | Opened a shell and read `/etc/shadow` inside a test container | Falco generated two alerts within a few milliseconds |
| CloudTrail | Checked my own security group change and EC2 launch | Both events were recorded and showed `root` as the user |



**Trivy numbers**

| Image | Total | Critical | High | Medium | Low |
|---|---:|---:|---:|---:|---:|
| nginx:1.27-alpine | 137 | 2 | 38 | 57 | 40 |

The `http-echo` scan contained 1 OS package finding and 71 findings in the Go binary. I haven't broken those down further yet.

![nigix alerts](evidence/nigix-report-before.png)

![nigix alerts](evidence/http-echo-before.png)


---

# Investigation 1: Shell and password file read inside a container

**Severity:** Medium  
**Detected by:** Falco  
**Date:** 2026-09-28, 11:08 UTC  
**Location:** Temporary Alpine pod (`falco-test`) in `security-lab`

For the first test, I created a temporary pod and ran a shell command that attempted to read `/etc/shadow`.

Falco generated two alerts:

- A shell was started inside a container with a terminal attached.
- A non-trusted program tried to read the sensitive `/etc/shadow` file.

Falco also showed the parent process (`containerd-shim`) and the container ID (`8d6bd396b35b`).

One interesting limitation was that Falco showed the container and pod name as `<NA>`. I matched the event back to my test using the command, container ID, and timing.

### What happened?

This was my own test, so there wasn't an actual attacker. In a real environment, though, the same behaviour could indicate someone gaining shell access to a container or someone who already has permission to create or access pods.

### What I would do

I deleted the temporary pod after the test.

In a real incident, I would isolate the affected workload, restrict its network access, remove the compromised pod if appropriate, and investigate who created or accessed it.

The main preventive controls would be tighter RBAC and avoiding containers running as root unless there is a specific reason to do so.

**MITRE ATT&CK:** T1059.004 (Unix Shell), T1003.008 (Security Account Manager)

![Falco alerts](evidence/falco-alerts.png)

---

# Investigation 2: Read-only account tries to delete things

**Severity:** Low  
**Detected by:** Kubernetes RBAC permission checks  
**Date:** 2026-09-28  
**Location:** `security-lab` namespace

For this test, I wanted to make sure the service accounts were actually restricted to the permissions I intended to give them.

I used `kubectl auth can-i` to check a few actions:

| Account | Action | Result |
|---|---|---|
| `analyst-sa` | Delete pods | No |
| `analyst-sa` | Get pods | Yes |
| `developer-sa` | Delete deployments | No |
| `developer-sa` | Create deployments | Yes |

The result was what I expected: the analyst can view pods but cannot delete them, while the developer can create deployments but cannot delete them.

This was a simple way of checking that the least-privilege rules were actually being enforced.

### Limitation

This wasn't a real attack. It was a permission check.

I also haven't configured Kubernetes audit logging yet, so I can't show a corresponding audit event identifying exactly who performed an action and when.

That's one of the next things I want to add to the lab.


![Rbac alerts](evidence/rbac-can-i.png)


---

# Investigation 3: Firewall change and EC2 launch recorded as root

**Severity:** Medium  
**Detected by:** AWS CloudTrail  
**Region:** `us-east-1`

This was probably the most useful finding in the lab because CloudTrail ended up catching something I did myself.

I checked the CloudTrail event history and found two actions from my testing:

| Event | Time (IST) | User | Source IP | Result |
|---|---|---|---|---|
| `AuthorizeSecurityGroupIngress` | 2026-09-28, 15:04:22 | root | My home IP | Success |
| `RunInstances` | 2026-09-28, 15:06:39 | root | My home IP | Success |

Both events were recorded as being performed by the `root` user using temporary session credentials from the console.

I had already intended to avoid using the root account for normal work, so seeing it show up in CloudTrail was a useful reminder of why that matters.

### What I would change

For a real AWS environment, I would:

- Avoid using the root account for day-to-day tasks.
- Enable MFA for the root account.
- Use a separate IAM identity with only the permissions required for normal work.
- Create an alert for root account activity.

**MITRE ATT&CK:** T1078.004 (Valid Accounts: Cloud Accounts), T1562.007 (Disable or Modify Cloud Firewall), T1578.002 (Create Cloud Instance)

![EC2 RunInstances event](evidence/run-instances.png)

![Security group change](evidence/authorizesecuritygroups.png)

---

---

# Problems I ran into

Not everything went smoothly. A few of the problems were completely unrelated to security, but fixing them was part of getting the lab running.

### K3s node became NotReady

The EC2 instance only had 1 GB of RAM, and K3s eventually ran out of memory.

I added a swap file and the node came back to `Ready`.

![K3s NotReady issue](evidence/not-ready.png)

### Trivy ran out of disk space

The original 8 GB disk filled up while working with the container images.

I increased the volume to 20 GB, after which Trivy was able to complete the scans.

![Disk space issue](evidence/no-space.png)

### SSH key permission error

Windows had extra users listed on the private key file, so SSH refused to use it.

I fixed the permissions with `icacls` and was able to connect normally.

![SSH key permission error](evidence/private-key-error.png)

---

# What I learned

A few things stood out while doing this lab.

- Installing a security tool isn't the same as actually using it. It became much more useful once I triggered something and could see the detection myself.
- Small cloud instances have practical limitations. Memory and disk space became problems before I even got deep into the security testing.
- CloudTrail made it very clear who performed an AWS action. In this case, it caught my own mistake of using the root account.
- Kubernetes Secrets are not automatically encrypted just because they're called "Secrets." The values I tested were base64-encoded, which is not encryption.
- Security controls are easier to understand when I actually test both sides — what should be allowed and what should be blocked.

---

## What's next

I still want to add Kubernetes audit logging and remediate some of the Trivy findings, then run the scans again to compare the results.
---