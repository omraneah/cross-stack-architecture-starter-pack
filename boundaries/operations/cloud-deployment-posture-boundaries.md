# Cloud Deployment Posture Boundaries

**Customer-facing services run behind a load balancer, with at least one healthy replica, on a deploy that doesn't drop connections. The absence of high availability at any of these layers is a conscious, documented trade-off — not a default.**

**Applies when:** customer-facing service (any service serving real users, partners, or paying customers in production). Skip for internal tooling, batch jobs, scheduled workers.

## Why it matters

- A single instance behind an Elastic IP is a single point of failure. Every deploy is a customer-visible outage; every instance failure is a customer-visible outage.
- A "deploy by shell script via SSH" is fragility — the script breaks, the SSH key gets rotated, the engineer who wrote it leaves, and now nobody can ship safely.
- No load balancer means no zero-downtime deploy, no autoscaling, no traffic shifting for canary or blue/green. Every release event is a coin flip.
- Firewalls and security groups left at defaults expose the cloud's full attack surface to the public internet.

## The judgment

**Deployment topology:**

- Load balancer (cloud-native or equivalent) in front of every customer-facing service. Default for anything past prototype.
- At least one healthy replica for production traffic: two instances behind an LB, or an autoscaling group, or a managed container service (Fargate, Kubernetes, Cloud Run). Single-instance production is acceptable only for explicit pre-launch / internal-tool contexts, time-boxed.
- Deploy that doesn't drop connections — rolling, blue/green, or canary. SSH-into-VM-and-restart-Docker is the fragility this boundary is named to prevent.

**Network posture:**

- Security groups / firewall rules scoped to least privilege. Public ingress only on the LB; internal services are not internet-reachable.
- Engineer access to private resources via session-manager or scoped allowlist; no direct SSH from the public internet to production instances.
- The IAM and access-control boundary covers the credential side; this boundary covers the network shape.

**Runtime substrate:**

- ECS / Fargate / Kubernetes / Cloud Run / autoscaling group — choose based on the team's operational depth. The choice is a real trade-off; the boundary's floor is "not a single VM with a deploy shell script for a customer-facing service past prototype."

## Signals of violation in an audited codebase

- A production service running on a single VM with no LB in front.
- A deploy workflow that SSHs into the instance and runs container restart commands.
- A health endpoint that always returns 200 regardless of dependency state (operational-integrity catches this from another angle; this boundary catches the deploy implication).
- Security groups allowing `0.0.0.0/0` ingress on application ports rather than only on the LB.
- No documented decision record stating why high availability is or is not in place.
- An IaC config that creates one instance with an Elastic IP and no LB resource.

## Minimum viable shape

Six independent rules over the deployment surface:

- **Customer-facing service:** behind a load balancer (cloud-native or equivalent).
- **Load balancer:** at least 2 healthy targets, an autoscaling group, or a managed container service.
- **Deploy:** rolling, blue/green, or canary; in-flight requests drain on termination.
- **Security groups:** least privilege; only the LB exposes public ingress on application ports.
- **Engineer access to private resources:** session manager or scoped allowlist; no internet-facing SSH.
- **Single-instance production:** explicit, documented, time-boxed; reviewed monthly.

**Severity floor if violated:** P0 for a customer-facing service without LB or running single-instance with SSH-script deploy. P1 for security group misconfiguration exposing application ports publicly. P2 for missing decision record on high availability.
