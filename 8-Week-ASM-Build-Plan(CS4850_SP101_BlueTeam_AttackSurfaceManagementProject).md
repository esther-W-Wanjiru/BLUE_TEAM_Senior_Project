# **8-Week ASM Build Plan**

---

## **Week 1 – Environment & Prerequisites**

* **Deliverables:**

  * Host machine (16GB RAM, 500GB+) provisioned.

  * Install base tools: `docker`, `docker-compose`, `python3`, `python3-venv`, `git`.

  * (Optional) Create dedicated `asm` user.

  * Git repo created for your project (scripts \+ docs).

* **Outcome:** Base host ready for ASM development.

---

## **Week 2 – Lab & Networking Setup**

* **Deliverables:**

  * Create two networks:

    * `network-external` (public-facing targets).

    * `network-internal` (internal services \+ ASM).

  * Deploy lab targets:

    * Ubuntu VM/container with a simple web app (Apache or Flask demo).

    * (Optional) Samba/FreeIPA for internal services.

  * Deploy ASM host (Ubuntu or Kali).

* **Outcome:** A working lab with isolated networks \+ test targets.

---

## **Week 3 – Deploy OpenSearch & Dashboards**

* **Deliverables:**

  * Create `docker-compose.yml` for OpenSearch \+ Dashboards.

  * Launch with `docker-compose up -d`.

  * Confirm accessibility:

    * [http://localhost:9200](http://localhost:9200) (OpenSearch API).

    * [http://localhost:5601](http://localhost:5601) (Dashboards).

* **Outcome:** Search & visualization backend is running.

---

## **Week 4 – Install ASM Tools**

* **Deliverables:**

  * Install tools on ASM host:  
     `nmap`, `masscan`, `amass`, `python3-pip`.

  * Install OpenVAS/GVM (either local package or Docker image).

  * Validate each tool works by running a sample scan.

* **Outcome:** Core ASM toolchain is installed and working.

---

## **Week 5 – Build Scan & Index Scripts**

* **Deliverables:**

  * Create Python venv (`~/asm-venv`).

  * Install libs: `opensearch-py`, `xmltodict`, `flask`, `apscheduler`, `gunicorn`.

  * Write `scan_and_index.py` (Nmap parser → OpenSearch index).

  * Extend later with Amass, Masscan, and OpenVAS parsers.

* **Outcome:** One-click Nmap scan results stored in OpenSearch (`asm-assets` index).

---

## **Week 6 – Develop Web Interface**

* **Deliverables:**

  * Flask app (`/opt/asm/webapp/app.py`) with pages:

    * `/` → Assets list.

    * `/vulns` → Vulnerabilities list.

    * `/scan` → On-demand scan trigger.

  * Create simple HTML templates using Bootstrap.

  * Run app locally (`flask run`).

* **Outcome:** Functional web UI to view assets and start scans.

---

## **Week 7 – Automation, Scheduling & Security**

* **Deliverables:**

  * Add **APScheduler** (or cron) to run scans every X hours.

  * Parse & index OpenVAS/GVM XML → `asm-vulns` index.

  * Secure the web app:

    * Run Flask with Gunicorn.

    * Configure Nginx reverse proxy with TLS (self-signed or Let’s Encrypt).

    * Add basic auth or Flask-Login.

* **Outcome:** Automated \+ secure system accessible at `https://asm.local` (or your domain).

---

## **Week 8 – Visualization, Testing & Final Demo**

* **Deliverables:**

  * Create OpenSearch Dashboards:

    1. Asset counts over time.

    2. Top CVSS vulnerabilities.

    3. Frequent open ports.

  * Test lab scenarios:

    1. Spin up a vulnerable web app and confirm detection.

    2. Add/remove assets and check discovery.

  * Create demo script:

    1. Run scan.

    2. Show assets/vulns in web UI.

    3. Show dashboards.

    4. Export sample report.

  * Backups \+ retention (logrotate, snapshot).

* **Outcome:** Capstone-ready demo with working website, dashboards, and reporting.

---

👉 By the end of **Week 8**, you’ll have:

* A **running ASM platform** (lab \+ tools \+ automation).

* A **web app dashboard** to trigger scans & view assets/vulns.

* **OpenSearch Dashboards** for metrics/visuals.

* A **demo workflow** for your capstone presentation.

---

Do you want me to also **map these 8 weeks into the SDLC phases** (Planning → Design → Implementation → Testing → Deployment → Maintenance)? That way you can present it formally for your capstone.

##### **You said:**

summerise into a few sentences : Week 2 – Lab & Networking Setup Deliverables: Create two networks: network-external (public-facing targets). network-internal (internal services \+ ASM). Deploy lab targets: Ubuntu VM/container with a simple web app (Apache or Flask demo). (Optional) Samba/FreeIPA for internal services. Deploy ASM host (Ubuntu or Kali). Outcome: A working lab with isolated networks \+ test targets.

###### **ChatGPT said:**

In Week 2, you’ll set up the lab and networking environment by creating two networks: one external for public-facing targets and one internal for services and the ASM host. You’ll then deploy target machines such as an Ubuntu web server and optionally Samba/FreeIPA for internal services, along with the ASM host (Ubuntu or Kali). The result is a functioning lab with isolated networks and test targets.

##### **You said:**

write in 1st persion

###### **ChatGPT said:**

In Week 2, I set up the lab and networking environment by creating two networks: one external for public-facing targets and one internal for services and the ASM host. I then deployed target machines, including an Ubuntu web server and optionally Samba/FreeIPA for internal services, along with the ASM host (Ubuntu or Kali). By the end of the week, I had a functioning lab with isolated networks and test targets.

