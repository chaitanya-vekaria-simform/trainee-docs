# 📚 DevOps & Cloud Engineering — Cheatsheets

> A curated collection of comprehensive cheatsheets for DevOps and Cloud Engineers.  
> Click into any category folder below to browse.

---

## 🗂️ Categories

| Category | Description |
|----------|-------------|
| [🪟 Windows](windows/) | Windows Server, Active Directory, GPO, DNS, DHCP, PowerShell |
| [🐧 Linux](linux/) | _Coming soon_ — Fundamentals, shell scripting, systemd, networking |
| [🌐 Servers](servers/) | Nginx, Apache, IIS — reverse proxy, SSL, static hosting, production Q&A |
| [🌐 Networking](networking/) | OSI/TCP-IP, subnetting, DNS, HTTP/TLS, firewalls, load balancers, VPNs, container networking |
| [☁️ Cloud](cloud/) | Data centers, virtualization, AWS/Azure/GCP, IaC, serverless, containers, DR, FinOps |
| [🧪 Labs](labs/) | Hands-on disaster recovery, troubleshooting, and infrastructure experiments |
| [📜 Scripting](scripting/) | Bash & Python scripting for DevOps — automation, CI/CD, K8s, monitoring |
| [☸️ Kubernetes](kubernetes/) | K8s architecture, workloads, networking, Helm, security, production patterns |
| [☁️ Azure](azure/) | Azure DevOps, AKS, Pipelines, IaC (Bicep/Terraform), monitoring, security, migration |
| [📊 Monitoring](monitoring/) | Prometheus, Grafana, Loki, ELK Stack — metrics, dashboards, log aggregation |
| [⚡ Power Platform](power-platform/) | Microsoft Power Automate — workflow automation, approvals, connectors |

---

## 📄 All Cheatsheets

| Cheatsheet | Category |
|------------|----------|
| [Windows Server 2022](windows/windows_server_2022_cheatsheet.md) | Windows |
| [Linux Admin Handbook](linux/linux_admin_handbook.md) | Linux |
| [Nginx](servers/nginx.md) | Servers |
| [Apache HTTP Server](servers/apache.md) | Servers |
| [IIS (Internet Information Services)](servers/iis.md) | Servers |
| [Computer Networking Handbook](networking/networking_handbook.md) | Networking |
| [Data Centers & Cloud Computing Handbook](cloud/cloud_handbook.md) | Cloud |
| [Linux DR — Rebuilding After `/etc` Deletion](labs/linux_etc_disaster_recovery.md) | Labs |
| [Nginx Config Recovery — PhotoRec File Carving](labs/nginx_config_recovery_photorec.md) | Labs |
| [Scripting for DevOps Handbook](scripting/scripting_handbook.md) | Scripting |
| [Kubernetes Handbook](kubernetes/kubernetes_handbook.md) | Kubernetes |
| [DevOps on Azure Handbook](azure/azure_handbook.md) | Azure |
| [Monitoring & Observability Stack Handbook](monitoring/monitoring_stack_handbook.md) | Monitoring |
| [Power Platform Automation Handbook](power-platform/power_platform_handbook.md) | Power Platform |

---

## 🏗️ Repo Structure

```
├── README.md              ← You are here
├── windows/
│   ├── README.md          ← Windows category index
│   └── windows_server_2022_cheatsheet.md
├── linux/
│   ├── README.md          ← Linux category index (coming soon)
│   └── linux_admin_handbook.md
├── networking/
│   ├── README.md          ← Networking category index
│   ├── networking_handbook.md  ← Comprehensive networking guide
│   └── networking_batch_state.md  ← Batch review tracker
├── cloud/
│   ├── README.md          ← Cloud category index
│   ├── cloud_handbook.md  ← Data centers & cloud computing guide
│   └── cloud_batch_state.md  ← Batch review tracker
├── servers/
│   ├── README.md          ← Servers category index
│   ├── PROGRESS.md        ← Batch writing progress tracker
│   ├── nginx.md           ← Nginx production guide
│   ├── apache.md          ← Apache HTTP Server production guide
│   └── iis.md             ← IIS production guide
├── tasks/
│   ├── networking/        ← Networking tasks (Level 1/2/3)
│   ├── cloud/             ← Cloud tasks (Level 1/2/3)
│   ├── linux/             ← Linux tasks
│   ├── nginx/             ← Nginx tasks
│   ├── apache/            ← Apache tasks
│   └── windows_server/    ← Windows Server tasks
└── labs/
    ├── README.md          ← Labs category index
    ├── networking_practice_lab.md  ← Multi-tier network lab
    ├── cloud_practice_lab.md  ← Production cloud infrastructure lab
    ├── apache_practice_lab.md
    ├── nginx_practice_lab.md
    ├── iis_practice_lab.md
    ├── linux_etc_disaster_recovery.md
    └── nginx_config_recovery_photorec.md
├── scripting/
│   ├── README.md          ← Scripting category index
│   ├── scripting_handbook.md  ← Bash & Python for DevOps guide
│   └── scripting_batch_state.md  ← Batch review tracker
├── kubernetes/
│   ├── README.md          ← Kubernetes category index
│   ├── kubernetes_handbook.md  ← Comprehensive K8s guide
│   └── kubernetes_batch_state.md  ← Batch review tracker
├── azure/
│   ├── README.md          ← Azure category index
│   ├── azure_handbook.md  ← DevOps on Azure guide
│   └── azure_batch_state.md  ← Batch review tracker
├── monitoring/
│   ├── README.md          ← Monitoring category index
│   └── monitoring_stack_handbook.md  ← Prometheus, Grafana, Loki, ELK guide
├── power-platform/
│   ├── README.md          ← Power Platform category index
│   └── power_platform_handbook.md  ← Power Automate guide
```

---

## 🤝 Adding a New Cheatsheet

1. Pick the right folder (`windows/`, `linux/`, or create a new category folder).
2. Add your `.md` file inside that folder.
3. Update the folder's `README.md` with a link to the new cheatsheet.
4. Update this root `README.md` table above.
5. Use standard markdown links — `[Display Text](relative/path.md)`.

---

> **Maintained by:** Vishvam Moliya
