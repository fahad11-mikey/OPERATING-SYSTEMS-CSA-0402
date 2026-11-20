# Operating Systems – Assignment 5 (CO5)
## Linux, Containers and Virtualisation for Efficient Operations

> **Evidence note:** Replace sample guidance with your own Linux, Docker, VM and Kubernetes commands, screenshots, measurements and cost estimates. Do not claim measurements you have not run.

## Case Study
The university Result Portal (Python web app + database) serves a few hundred users on normal days but receives 150,000 logins within two hours on result day. The current single Linux server crashes. Recommended design: Linux baseline, containers, horizontal scaling and a hybrid cloud capacity burst with monitoring and backups.

# PART A – DESIGN

## A1 – Evaluate Linux System Components (7 Marks)

### Linux architecture
- **Kernel:** manages CPU scheduling, memory, processes, devices, filesystems and networking.
- **System libraries:** provide APIs that applications use to request kernel services.
- **Shell:** accepts commands and runs scripts.
- **Utilities:** tools for monitoring, files, users, networking and administration.

### Process and memory management
Linux's Completely Fair Scheduler (CFS) distributes CPU time fairly among runnable tasks. The page cache keeps frequently accessed file data in RAM to reduce disk I/O. Swap extends memory capacity but is much slower than RAM and can cause thrashing. `/proc` exposes process/kernel information; `/sys` exposes device and kernel configuration information.

### Filesystem and permissions
Linux directories include `/`, `/home`, `/etc`, `/var`, `/usr`, `/tmp`, `/dev`, `/proc` and `/sys`. VFS provides a common interface to filesystem types. Permissions are owner/group/others with read/write/execute bits. `sudo` runs approved commands with elevated privileges; setuid programs execute with the file owner's effective ID and must be used carefully.

### Windows Server alternative
Windows Server offers familiar GUI administration and strong Microsoft ecosystem integration, but its base resource use and licensing can be higher depending on edition and configuration. Linux is suitable for a Python portal, efficient, scriptable and generally avoids per-server OS licence cost. Compare actual installed footprint, admin automation time and current licence quotes before choosing.

## A2 – System Administration Design (5 Marks)

### Users, services and logs
Create separate accounts such as `portal`, `dbadmin` and administrators; use groups for controlled access. Run the portal through a systemd unit with restart policy, a dedicated non-root user and environment/config files with restricted permissions. Monitor logs using `journalctl` and rotate application logs.

### Network design
- Private server IP, gateway and subnet; reserve addresses through DHCP or static configuration.
- DNS name such as `results.university.edu` points to the load balancer/reverse proxy.
- Nginx terminates HTTPS and forwards requests to the app.
- Firewall allows only SSH from admin networks, HTTP redirect and HTTPS publicly; deny other inbound traffic.
- SSH uses keys; disable password login where appropriate and restrict administrators.

### Automation
Automate database backups, log rotation, updates, disk-space checks and alerts using shell scripts and cron. Example hourly schedule:
```cron
0 * * * * /usr/local/bin/disk-alert.sh
```

**Ansible alternative:** shell scripts are quick for one host, but Ansible can configure 20 servers consistently and reduce drift. Initial learning/playbook setup takes longer; repeat deployment is faster and more auditable. For one server, scripts may be simplest; for many servers, configuration management is safer.

## A3 – Evaluate Virtualisation (5 Marks)

### Hypervisors
- **Type 1:** runs directly on hardware (KVM, ESXi, Xen, Hyper-V); generally preferred for production.
- **Type 2:** runs as an application on a host OS (VirtualBox, VMware Workstation); convenient for desktop testing.
- **Full virtualisation:** guest OS behaves as if on hardware.
- **Para-virtualisation:** guest uses hypervisor-aware drivers/interfaces for efficiency.
Hardware extensions include Intel VT-x and AMD-V.

### Benefits and costs
VMs isolate services, consolidate servers, support snapshots and live migration. Costs include CPU/I/O overhead, RAM for each guest OS, management effort and possible licensing.

**Type-2 alternative:** easier to set up for lab/testing, but host OS adds resource overhead. CPU-bound workloads may be reasonably efficient with hardware support; I/O-heavy workloads can suffer extra overhead. Not ideal for production result-day load compared with a type-1 hypervisor.

## A4 – Evaluate Containerisation (4 Marks)

### How containers work
Containers use Linux namespaces to isolate process, network, mount and other views. cgroups control CPU, memory and other resource usage. Images are built from layered filesystems; each layer can be cached and reused.

### Containers vs VMs
| | Containers | Virtual machines |
|---|---|---|
| Start-up | Seconds or less | Often minutes |
| Memory | Share host kernel; lighter | Guest OS per VM |
| Isolation | Process/kernel features | Separate guest kernel; stronger boundary |
| Use | Scalable app replicas | Different OS, stronger isolation |

Use containers for portal app replicas; use VMs for database isolation or services needing a separate OS/security boundary. If app and database ran in separate VMs, isolation is stronger but start-up is slower and memory overhead increases because each VM has its own guest OS.

## A5 – Cloud Deployment and Orchestration (4 Marks)

### Cloud service models
- **IaaS:** virtual machines, networks and storage managed by the user.
- **PaaS:** provider manages runtime/platform; user deploys application.
- **SaaS:** complete application delivered as a service.

For result day, use cloud VMs or managed Kubernetes, load balancer, autoscaling portal replicas and managed database with backups/private access. Hybrid: keep baseline services on university infrastructure and burst to cloud when traffic rises.

### Kubernetes
- **Pod:** smallest deployable unit containing container(s).
- **Deployment:** manages desired replicas and replaces failed pods.
- **Service:** stable network endpoint and load distribution.
- **Replicas:** duplicate instances for capacity/availability.
- **Requests/limits:** scheduling reservation and resource ceilings.
- **HPA:** adjusts replica count based on observed metrics.

Fixed-size VMs can meet peak demand but waste resources during the other 364 days and may require manual scaling. Autoscaling responds to demand and can reduce idle cost, though scaling and cloud costs require monitoring.

# PART B – IMPLEMENTATION & EXPERIMENTAL INVESTIGATION

## B1 – Administer the Linux Server (7 Marks)

### Observe
```bash
ps aux
 top
 free -h
 vmstat 1 5
 iostat -xz 1 5
 df -h
 ss -tulpn
```
Use these to observe processes, CPU, memory, paging, disk activity, free disk space and listening sockets. Install `sysstat` if `iostat` is missing.

### Log-analysis script (completed TODOs)
```bash
#!/usr/bin/env bash
set -euo pipefail
log=$(mktemp)
cat > "$log" <<'LOG'
10.0.0.5 - - [08/Oct/2026:10:01:02] "GET /index.html" 200 512
10.0.0.7 - - [08/Oct/2026:10:01:05] "GET /login" 200 1024
10.0.0.5 - - [08/Oct/2026:10:01:09] "POST /login" 302 0
10.0.0.9 - - [08/Oct/2026:10:02:11] "GET /admin" 403 128
10.0.0.5 - - [08/Oct/2026:10:02:15] "GET /report.pdf" 200 40960
10.0.0.7 - - [08/Oct/2026:10:03:01] "GET /missing" 404 64
10.0.0.5 - - [08/Oct/2026:10:03:30] "GET /index.html" 200 512
LOG
echo "Requests per status code:"
awk '{print $7}' "$log" | sort | uniq -c | awk '{print $2, $1}' | sort -n
echo "Busiest client:"
awk '{print $1}' "$log" | sort | uniq -c | sort -nr | head -1 | awk '{print "  " $2, $1}'
echo "Total bytes sent: $(awk '{s += $8} END {print s}' "$log")"
rm -f "$log"
```
Expected counts: status 200 = 4, 302 = 1, 403 = 1, 404 = 1; busiest client `10.0.0.5` = 4; total bytes = 43200. Save script and run `bash log-summary.sh`. Hourly cron line:
```cron
0 * * * * /path/to/log-summary.sh >> /var/log/log-summary.out 2>&1
```

### Services/network and comparison
Use a systemd unit to start the sample app, Nginx reverse proxy and firewall configuration; only expose SSH/HTTP/HTTPS as specified. Compare the implemented settings with Part A and record any differences. Central logging (journald forwarding/ELK) makes searching errors across 10 servers faster but needs storage, RAM, setup, network and maintenance.

## B2 – Containerise the Portal (5 Marks)

### Dockerfile (TODOs completed)
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN useradd --create-home portal && chown -R portal:portal /app
USER portal
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```
Build and run:
```bash
docker build -t result-portal .
docker run -p 8000:8000 result-portal
docker ps
docker images
docker exec <container> whoami
docker stats
docker history result-portal
```
`whoami` should report `portal`. Record the image size and actual resource values.

### Docker Compose with database volume (example)
```yaml
services:
  portal:
    build: .
    ports: ["8000:8000"]
    depends_on: [db]
    mem_limit: 512m
    cpus: 0.5
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: change-this-password
      POSTGRES_DB: portal
    volumes:
      - dbdata:/var/lib/postgresql/data
volumes:
  dbdata:
```
Use secrets/environment variables securely; do not commit real passwords. A slim base image and multi-stage build can reduce image size, pull time and packages that may contain vulnerabilities. A full Ubuntu base is larger and includes more packages, but may provide familiar tooling. Compare build time, size and pull time with your actual measurements.

## B3 – Bare Metal, VM and Container Comparison (8 Marks)

### Deploy and measure
Run the same app on Linux directly, in a VM, and in Docker. Measure:
- startup time;
- memory use;
- disk/image size;
- requests per second using `ab`, `wrk` or `hey`.

| Deployment | Startup | Memory | Size | Requests/sec |
|---|---|---|---|
| Bare metal | Measure | Measure | App/runtime | Measure |
| VM | Measure | Measure | VM disk | Measure |
| Container | Measure | Measure | Image | Measure |

Use identical test conditions and repeat runs. Bare metal generally has lowest virtualization overhead; VMs have stronger isolation but more overhead; containers are lightweight and fast to start but share the host kernel.

### Kubernetes deployment (TODOs completed)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result-portal
spec:
  replicas: 3
  selector:
    matchLabels:
      app: result-portal
  template:
    metadata:
      labels:
        app: result-portal
    spec:
      containers:
        - name: portal
          image: simats/result-portal:1.0
          ports:
            - containerPort: 8000
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: result-portal
spec:
  type: LoadBalancer
  selector:
    app: result-portal
  ports:
    - port: 80
      targetPort: 8000
```
Apply and verify:
```bash
kubectl apply -f deployment.yaml
kubectl get pods
kubectl describe pod <pod-name>
kubectl delete pod <pod-name>
kubectl get pods -w
kubectl scale deployment result-portal --replicas=6
```
A Deployment recreates deleted pods to maintain the desired replica count. Docker Compose scaling on one server is simpler but cannot provide multi-node scheduling or robust server-failure recovery like Kubernetes; capacity is limited to the host.

## B4 – Final Engineering Decision (5 Marks)

### Practical extension
Add a second service (for example a learning portal) with its own container and CPU/memory limits. Verify both services are running and record `docker stats` or Kubernetes resource output.

### Capacity estimate
150,000 users in 2 hours means average arrivals:
`150,000 / 7,200 = 20.83 users/second`.
Use measured requests/second per container to estimate replicas:
`required replicas = ceil(target requests/second / sustainable requests/second per replica)`.
Add headroom for peaks, database limits, session handling and retries. Login arrivals are not necessarily equal to requests/sec; measure workload behavior.

### Production choice
Recommend a **hybrid deployment**: maintain baseline capacity on university infrastructure with containers/Kubernetes, use cloud autoscaling/load balancing for result-day burst, and use a managed or replicated database with backups, monitoring and private network access. This avoids buying peak-only servers while retaining control of baseline systems. Validate costs, data residency, security and failover using measured load.

### Public-cloud-only alternative
Cloud-only capacity is quick to provision and pay-per-use, but requires yearly cost calculation: baseline hours + two result days per semester (depending on academic calendar) + storage, network and managed database costs. Cloud provides provider-managed infrastructure; the university remains responsible for data governance, access, configuration and compliance. Record actual provider pricing and estimates before concluding.

---

## GitHub Submission Checklist
- [ ] Repository: `CSA04-<RegisterNumber>-Assignment5`
- [ ] `README.md` with student details, compiler, Linux version, commands and results
- [ ] Separate folders for Task A1–A5 and B1–B4
- [ ] Scripts, C/Python source, Dockerfile, Compose YAML and Kubernetes YAML
- [ ] Outputs, screenshots and measurements
- [ ] Progressive commits (at least one per task)
- [ ] Working repository link included in report
