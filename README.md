# Assignment_10_DevOps

# End‑to‑End MERN Application Deployment & Monitoring (AWS + Terraform + Ansible + Prometheus + Grafana)

This repository automates provisioning, deployment, and monitoring of the **TravelMemory** MERN app on AWS using **Terraform** (infra), **Ansible** (configuration), **Prometheus** (metrics), and **Grafana** (dashboards & alerts).

> App upstream: [https://github.com/UnpredictablePrashant/TravelMemory](https://github.com/UnpredictablePrashant/TravelMemory)

---

## 📁 Repository Layout (actual files)

```
ASSIGNMENT_10_DEVOPS/
├─ ansible/
│  ├─ dbserver.yml           # Configure MongoDB DB node
│  ├─ webserver.yml          # Configure Node/React + Prometheus + Grafana
│  ├─ inventory.ini          # Filled from Terraform outputs
│
├─ monitoring/
│  └─ prometheus.yml         # Prometheus scrape config (web + mongo-exporter)
│
├─ terraform/
│  ├─ main.tf                # Provider, networking, SGs, EC2, IAM (as needed)
│  └─ outputs.tf             # Prints public IPs of web & db
│
├─ azure-pipelines.yml       # (Optional) Example CI entrypoint
├─ .gitignore
├─ LICENSE
└─ README.md                 # You are here
```

---

## 🧭 High‑Level Architecture

* **VPC**: Default VPC (public subnet)
* **EC2**: two instances

  * **Web/App**: Node.js backend + React frontend, **Prometheus** & **Grafana** colocated
  * **DB**: MongoDB + **mongodb‑exporter** (for Prometheus)
* **Security**

  * Web SG: Inbound HTTP/HTTPS/SSH from the internet
  * DB SG: Inbound 27017 **only** from Web SG + SSH from your IP
* **Observability**

  * Backend exposes custom metrics via `prom-client` (HTTP `/metrics`)
  * MongoDB metrics via `mongodb-exporter`
  * Prometheus scrapes web + db; Grafana visualizes and alerts

```
Internet → [ALB/Direct :80/:443] → Web EC2 ────► MongoDB EC2
                         │
                         ├── Prometheus (scrapes web + mongo-exporter)
                         └── Grafana (dashboards & alerts)
```

---

## ✅ Prerequisites

* AWS account with IAM user/role able to create EC2, SG, IAM roles
* Locally installed: `awscli`, `terraform` (>=1.4), `ansible` (>=2.14), `ssh`
* A key pair in AWS (or generate one locally and import)

---

## 1) Provision Infrastructure with Terraform

### Configure AWS

```bash
aws configure  # or use environment variables for CI
```

### Initialize & Apply

```bash
cd terraform
terraform init
terraform plan -out tf.plan
terraform apply tf.plan
```

### Expected Terraform Outputs

`terraform/outputs.tf` prints the **public IPs** you will need for Ansible:

```
web_public_ip = 13.XX.XX.XX
mongodb_public_ip = 3.XX.XX.XX
```

> Copy these into `ansible/inventory.ini` (next step).

> **Note:** Both instances are created in the default VPC’s public subnet, with the Web SG allowing `80/443/22` and DB SG allowing `22` from your IP and `27017` from Web SG only.

---

## 2) Configure & Deploy with Ansible

### 2.1 Inventory

Edit `ansible/inventory.ini` using the IPs from Terraform:

```ini
[web]
web1 ansible_host=WEB_PUBLIC_IP ansible_user=ubuntu

[db]
db1 ansible_host=DB_PUBLIC_IP ansible_user=ubuntu

[all:vars]
ansible_ssh_private_key_file=~/.ssh/your-key.pem
app_repo=https://github.com/UnpredictablePrashant/TravelMemory
node_env=production
mongo_port=27017
```

### 2.2 Sample `.env` for the app (checked‑in as example with secrets redacted)

Create on the **web server** under backend & frontend after clone; Ansible playbooks will template these values:

```dotenv
# backend/.env
PORT=8080
MONGO_URI=mongodb://appuser:REDACTED_PASSWORD@DB_PRIVATE_OR_PUBLIC_IP:27017/travel_memory?authSource=admin
JWT_SECRET=REDACTED
PROMETHEUS_METRICS_PORT=9101

# frontend/.env
REACT_APP_API_BASE_URL=http://WEB_PUBLIC_IP:8080
```

> Replace `REDACTED_*` and IPs accordingly. Do **not** commit secrets.

### 2.3 Run DB server play (MongoDB + exporter)

```bash
cd ansible
ansible-playbook -i inventory.ini dbserver.yml
```

What it does:

* Installs & configures **MongoDB**
* Creates DB `travel_memory`, user `appuser` with app‑scoped role
* Enables remote bind (`0.0.0.0`) but relies on SG restrictions for security
* Installs & starts **mongodb‑exporter** (default on `:9216`)

### 2.4 Run Web server play (Node/React + Prometheus + Grafana)

```bash
ansible-playbook -i inventory.ini webserver.yml
```

What it does:

* Installs **Node.js** + **npm**
* Clones `TravelMemory` repo and installs `backend/` and `frontend/` deps
* Creates `.env` files from templates
* Adds basic **prom-client** metrics to backend and starts it (PM2/systemd/nohup – as implemented in playbook)
* Builds and serves **React** frontend
* Installs **Prometheus** (reads `monitoring/prometheus.yml`) and **Grafana**

---

## 3) Observability

### 3.1 Backend metrics (prom-client)

The playbook configures the backend to expose metrics at `http://WEB_PUBLIC_IP:9101/metrics` and adds counters/histograms for:

* **Request count**
* **Error rate**
* **API latency** (histogram)

> If you prefer doing it manually, add `prom-client` to the backend and publish `/metrics` via Express. Ensure the port matches `PROMETHEUS_METRICS_PORT`.

### 3.2 MongoDB exporter

Runs on the DB instance (default `:9216`). Prometheus scrapes it using the job label `mongodb`.

### 3.3 Prometheus configuration

See `monitoring/prometheus.yml`. Example targets (replace with your IPs):

```yaml
scrape_configs:
  - job_name: 'node-backend'
    static_configs:
      - targets: ['WEB_PUBLIC_IP:9101']

  - job_name: 'mongodb'
    static_configs:
      - targets: ['DB_PUBLIC_IP:9216']
```

Start/verify on the web server: `http://WEB_PUBLIC_IP:9090`

### 3.4 Grafana

* URL: `http://WEB_PUBLIC_IP:9095` (default set by playbook)
* Default credentials: `admin / admin` (forced change on first login)
* Add Prometheus data source (URL: `http://localhost:9090`)
* Import dashboards:

  * **Custom Backend Metrics** (panel JSON provided by playbook or build manually)
  * **MongoDB Overview** (use community dashboards or create panels for `mongodb_*` series)

### 3.5 Alerts (Grafana)

Create alert rules for:

* **High error rate**: backend `http_requests_total{status=~"5.."}`
* **Slow API**: latency histogram p95 above threshold for N minutes
* **MongoDB health**: e.g., `mongodb_up == 0` or connections/s > threshold

Hook up contact points (Email/SMTP already supported; Slack/Teams optional). The playbook can template basic notifier config if SMTP creds are provided as vars.

---

## 4) Runbook / How to Operate

### Re‑run Terraform (change size, tags, etc.)

```bash
cd terraform
terraform plan
terraform apply
```

### Re‑provision servers (idempotent)

```bash
cd ansible
ansible-playbook -i inventory.ini dbserver.yml
ansible-playbook -i inventory.ini webserver.yml
```

### Where things run

* Backend API: `http://WEB_PUBLIC_IP:8080`
* Frontend:    `http://WEB_PUBLIC_IP` (or the port configured by your web play)
* Prometheus:  `http://WEB_PUBLIC_IP:9090`
* Grafana:     `http://WEB_PUBLIC_IP:9095`
* Metrics:

  * Backend:   `http://WEB_PUBLIC_IP:9101/metrics`
  * MongoDB:   `http://DB_PUBLIC_IP:9216/metrics`

> If you place an Nginx/ALB in front later, update the URLs accordingly.

---

## 5) Security Notes

* **DB SG** only allows `27017` from the Web SG; no world‑open DB
* SSH hardened by Ansible: password login off, key‑only, optional root login disabled
* Do not commit real secrets; store in Ansible Vault or CI secrets

---

## 6) Troubleshooting

* **`terraform destroy` fails subnet/SG dependency**: detaches IGW and deletes dependent resources first (Terraform handles order; see errors for blockers)
* **Prometheus cannot scrape targets**: open SGs/ports, verify `prometheus.yml` targets and that services are listening
* **Grafana empty panels**: confirm Prometheus datasource URL and scrape jobs; check series exist in `http://WEB_PUBLIC_IP:9090/graph`
* **MongoDB auth errors**: validate user/role and `authSource=admin` in `MONGO_URI`

---

## 7) Deliverables Checklist

* [x] **terraform/** code with provider, SGs, EC2, outputs
* [x] **ansible/** playbooks (`webserver.yml`, `dbserver.yml`, `inventory.ini`)
* [x] **monitoring/** Prometheus config (`prometheus.yml`)
* [x] `docs/` (optional) with architecture diagram & screenshots
* [x] Sample `.env` (secrets redacted)
* [x] Screenshots: Grafana dashboards & alert rules

---

## 8) Commands Quick Reference

```bash
# 1) Infra
cd terraform && terraform init && terraform apply -auto-approve

# 2) Inventory
cd ../ansible && $EDITOR inventory.ini

# 3) Configure DB then Web
ansible-playbook -i inventory.ini dbserver.yml
ansible-playbook -i inventory.ini webserver.yml

# 4) Verify endpoints
curl http://WEB_PUBLIC_IP:8080/health
curl http://WEB_PUBLIC_IP:9101/metrics
curl http://DB_PUBLIC_IP:9216/metrics
```

---

## 9) Performance & Observability Tips

* Track **p95 latency** and **error rate** per route; alert on short windows (3–5m) with 2–3 eval cycles
* Watch **MongoDB connections**, **opcounters**, and **locks**; correlate with API latency
* Use **instance tags** and **job labels** in Prometheus for clean dashboards

---

## 10) Future Enhancements

* Put an **ALB** in front of the web server and use proper domain + TLS (ACM)
* Containerize & move to **EKS**; use **Helm** & **Ingress‑NGINX**
* Centralized logs with **CloudWatch** or **Loki**
* Infrastructure hardening (private subnets + NAT, SSM Session Manager, backups)

---

## 📸 Screenshots:

<img width="545" height="130" alt="image" src="https://github.com/user-attachments/assets/b2b7815d-65f2-4bac-b20d-ba2432610e51" />
<img width="1621" height="412" alt="image" src="https://github.com/user-attachments/assets/b6b24f9c-f09b-4873-bc61-2bad9343701c" />
<img width="1637" height="355" alt="image" src="https://github.com/user-attachments/assets/92becc7e-ca54-46ff-b50c-63890467b76d" />
<img width="1032" height="56" alt="image" src="https://github.com/user-attachments/assets/6c2864f9-0a7e-4520-9697-d93ead1879ed" />
<img width="1918" height="965" alt="image" src="https://github.com/user-attachments/assets/7d1e522c-7a81-485c-9d2e-cf96dc778855" />
<img width="1918" height="956" alt="image" src="https://github.com/user-attachments/assets/8203dd9c-6ba9-4c79-960b-eba007406e92" />
<img width="323" height="101" alt="image" src="https://github.com/user-attachments/assets/962bf090-fcd7-4879-b0ad-935819e9efcd" />
<img width="1042" height="447" alt="image" src="https://github.com/user-attachments/assets/947edd75-86b5-425f-a5e3-09ae803e42a1" />
<img width="1000" height="402" alt="image" src="https://github.com/user-attachments/assets/ee398171-0219-4a4a-a1a0-dca54a28434c" />
<img width="907" height="372" alt="image" src="https://github.com/user-attachments/assets/2d748f28-d20f-455d-a45a-e035fd798723" />
<img width="565" height="152" alt="image" src="https://github.com/user-attachments/assets/09303cd6-e44e-49d1-b3bf-ca4e15e0c32d" />

---

## 📜 License
This project is licensed under the MIT License.

## 🤝 Contributing
Feel free to fork and improve the scripts! ⭐ If you find this project useful, please consider starring the repo—it really helps and supports my work! 😊

## 📧 Contact
For any queries, reach out via GitHub Issues.

---

🎯 **Thank you for reviewing this project! 🚀**
