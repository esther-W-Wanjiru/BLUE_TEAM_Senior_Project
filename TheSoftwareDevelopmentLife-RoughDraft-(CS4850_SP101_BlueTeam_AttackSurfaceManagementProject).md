# **Attack Surface Management (ASM) — SDLC Document**

## **1\) Planning phase & Scope**

**Goal.** Build a web-accessible, open-source ASM platform that continuously discovers assets, identifies vulnerabilities, and visualizes risk.  
 **In-scope.** Virtual lab assets (Ubuntu web servers, internal services), discovery and vuln scanning (Nmap, Amass, Masscan, OpenVAS/GVM), data indexing/visualization (OpenSearch \+ Dashboards), web UI (Flask), scheduling (cron/APScheduler), secure access (Nginx \+ TLS).  
 **Out-of-scope.** Production internet scanning; commercial tools; enterprise ticketing integrations.  
 **Success Metrics.**

* Discovery coverage ≥ 95% of seeded lab assets.

* Time-to-detection for new asset ≤ 1 scan interval.

* Vulnerabilities indexed with CVE/CVSS fields and visible in UI.

* Secure HTTPS access with basic auth.

## **2\) Requirements**

### **2.1 Functional Requirements (FR)**

* **FR1**: Discover hosts/services on defined networks (Nmap/Masscan).

* **FR2**: Enumerate domains/subdomains (Amass) for external-facing scope.

* **FR3**: Run vulnerability scans (OpenVAS/Nikto/Wapiti).

* **FR4**: Parse scanner outputs and index to OpenSearch (`asm-assets`, `asm-vulns`).

* **FR5**: Provide a web UI to list assets, view vulnerabilities, and trigger on-demand scans.

* **FR6**: Schedule recurring scans (cron/APScheduler).

* **FR7**: Provide dashboards (OpenSearch Dashboards) for trends/metrics.

* **FR8**: Protect web UI with authentication and TLS via Nginx reverse proxy.

### **2.2 Non-Functional Requirements (NFR)**

* **NFR1 Performance**: Support scanning a /24 in ≤ 1 hour with default profiles (lab hardware).

* **NFR2 Reliability**: Survive service restarts without data loss (Docker volumes).

* **NFR3 Security**: HTTPS enforced; OpenSearch not exposed publicly; least privilege.

* **NFR4 Usability**: Web UI lists assets/vulns within 5 seconds for ≤ 500 docs.

* **NFR5 Openness**: 100% open-source components.

## **3\) High-Level Architecture (HLA)**

Components (maps to your overview):

1. **Lab Targets**: Ubuntu web servers (Dockerized apps), optional Samba/FreeIPA internal services.

2. **ASM Host**: Runs Nmap, Masscan, Amass, OpenVAS/GVM, parsers, scheduler, Flask UI.

3. **Indexing/Visualization**: OpenSearch \+ OpenSearch Dashboards.

4. **Web Access Layer**: Nginx reverse proxy \+ TLS.

5. **Networks**: `network-external` (public-facing), `network-internal` (internal \+ ASM reach).

## **4\) Design**

### **4.1 Data Model (OpenSearch)**

* **Index `asm-assets`**: `{ ip, ports:[{port,protocol,state,service}], source, first_seen, last_seen }`

* **Index `asm-vulns`**: `{ host_ip, service, cve, cvss, severity, description, scanner, first_seen }`

* **Linking**: `asm-vulns.host_ip → asm-assets.ip`.

### **4.2 API / Web UI**

* Routes: `/` (Assets), `/vulns` (Vulnerabilities), `/scan` (POST start scan).

* Scan worker spawns `scan_and_index.py` (async thread).

* AuthN via Nginx basic auth or Flask-Login (capstone: basic auth acceptable).

### **4.3 Security Design**

* Reverse proxy terminates TLS; only Nginx exposed on 443\.

* OpenSearch bound to local network; security plugin can remain disabled for single-node lab, but restrict access (UFW/VPN).

* Principle of least privilege for services; non-root containers where possible.

## **5\) Implementation**

### **5.1 Environment & Prereqs**

* **Hardware**: x86\_64, 16 GB RAM, ≥ 500 GB disk.

* **Base software**: Docker/Compose, Python 3, venv, git.

* **(Optional)** Create user: `sudo adduser asm`.

### **5.2 Networking & Lab Setup**

* Create VirtualBox or Docker networks: `network-external`, `network-internal`.

* Provision VMs/containers: ASM host, target web servers (Docker), OpenSearch stack, optional Samba/FreeIPA.

### **5.3 Indexing/Visualization Backend (Docker Compose)**

See **Appendix A** for `docker-compose.yml` (OpenSearch \+ Dashboards).  
 Bring up: `docker-compose up -d`, verify `:9200`, `:5601`.

### **5.4 Tooling on ASM Host**

`sudo apt update`  
`sudo apt install -y nmap masscan amass python3-pip python3-venv git`

OpenVAS/GVM via distro packages or trusted Docker image; schedule scans and export XML.

### **5.5 Automation Scripts (Discovery → Index)**

* Create Python venv; install deps: `opensearch-py xmltodict flask apscheduler gunicorn`.

* Implement **`scan_and_index.py`** for Nmap XML → `asm-assets`.

* Add parsers for Amass/Masscan/OpenVAS to also populate `asm-vulns`.

### **5.6 Web Application (Flask)**

* Routes `/`, `/vulns`, `/scan` with async scan trigger.

* Templates render tables (Bootstrap).

* Dev run: `flask run --host=0.0.0.0 --port=5000`.

* Prod: Gunicorn behind Nginx.

### **5.7 Scheduling**

* APScheduler job inside app (or `cron`) to run CIDR scans every N hours.

## **6\) Testing Strategy**

### **6.1 Unit Tests**

* Parser correctness (Nmap/OpenVAS XML → JSON docs).

* Field validation (must include `ip`, `cvss` when applicable).

### **6.2 Integration Tests**

* End-to-end: trigger `/scan` → docs appear in `asm-assets`.

* OpenVAS export → `asm-vulns` ingestion.

### **6.3 System/Acceptance Tests**

* **Discovery accuracy** (≥ 95% of seeded assets).

* **Detection timeliness** (≤ interval).

* **UI responsiveness** (\< 5s for 500 rows).

* **Security**: HTTPS enforced; no OpenSearch public access.

## **7\) Deployment**

### **7.1 Reverse Proxy & TLS**

* Nginx proxies Gunicorn (`127.0.0.1:8000`) and enforces HTTPS.

* Let’s Encrypt (public) or self-signed (lab).

* Example server block in **Appendix C**.

### **7.2 Ports & Firewall**

* Allow 22, 443 (and 5601 only if you choose to expose Dashboards).

* Restrict OpenSearch to localhost or VPN network.

## **8\) Operations & Maintenance**

* OpenSearch snapshots / index lifecycle; log rotation (`logrotate`).

* Regular updates of tools and OS packages.

* Back up exported reports and config (Compose files, parsers, app).

## **9\) Risk Management**

* **Network disruption from scans** → throttle, test windows, exclude fragile hosts.

* **Parser failures** → schema validation, unit tests, fallback logging.

* **Data exposure** → keep OpenSearch private; mask sensitive fields; HTTPS.

* **Tool CVEs** → schedule apt/docker image updates.

## **10\) Project Timeline (example — 8 weeks)**

* **Wk 1**: Planning, scope, success metrics, lab networks.

* **Wk 2**: OpenSearch stack up; target apps deployed.

* **Wk 3**: Install Nmap/Masscan/Amass; Nmap parser to `asm-assets`.

* **Wk 4**: OpenVAS deploy; vuln parser to `asm-vulns`.

* **Wk 5**: Flask UI (assets, vulns, `/scan`).

* **Wk 6**: Scheduling; dashboards; basic auth \+ TLS.

* **Wk 7**: Testing, tuning, metrics, backups.

* **Wk 8**: Final hardening, documentation, demo.

## **11\) Deliverables**

* Running **web site** (HTTPS) with asset/vuln views and on-demand scans.

* OpenSearch Dashboards with trends/metrics.

* Source repo (Compose files, parsers, Flask app).

* Test/Evaluation report (coverage, timeliness, MTTD, false positives).

* Operations guide (backup, restore, update).

## **12\) Acceptance Criteria (Definition of Done)**

* DoD-1: Assets from both networks appear in `asm-assets` with ports/services.

* DoD-2: At least one OpenVAS finding indexed with CVE/CVSS in `asm-vulns`.

* DoD-3: UI can trigger a scan and show new results without restart.

* DoD-4: HTTPS only; basic auth required.

* DoD-5: Dashboard shows “Top CVSS,” “Assets over time,” “Open ports by service.”

* DoD-6: Documentation enables a clean redeploy on a new host.

## **13\) Compliance & Ethics**

* Scan only lab assets you own/have permission for.

* Avoid internet-wide scanning; throttle to prevent disruption.

* Clearly label demo data and environments.

---

## **Appendix A — Docker Compose (OpenSearch \+ Dashboards)**

`version: '3.8'`  
`services:`  
  `opensearch:`  
    `image: opensearchproject/opensearch:2.10.0`  
    `container_name: opensearch`  
    `environment:`  
      `- discovery.type=single-node`  
      `- OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m`  
      `- plugins.security.disabled=true`  
    `ulimits:`  
      `memlock: { soft: -1, hard: -1 }`  
    `volumes:`  
      `- opensearch-data:/usr/share/opensearch/data`  
    `ports:`  
      `- "9200:9200"`

  `dashboards:`  
    `image: opensearchproject/opensearch-dashboards:2.10.0`  
    `container_name: opensearch-dashboards`  
    `environment:`  
      `- OPENSEARCH_HOSTS=http://opensearch:9200`  
    `ports:`  
      `- "5601:5601"`  
    `depends_on: [opensearch]`

`volumes:`  
  `opensearch-data:`

## **Appendix B — Key Scripts (Discovery → Index)**

Create venv and install:

`python3 -m venv ~/asm-venv`  
`source ~/asm-venv/bin/activate`  
`pip install opensearch-py xmltodict flask apscheduler gunicorn`

**`/opt/asm/scan_and_index.py`** (Nmap XML → `asm-assets`):

`#!/usr/bin/env python3`  
`import subprocess, tempfile, xmltodict, os`  
`from opensearchpy import OpenSearch`

`OPENSEARCH = {'host':'localhost','port':9200}`

`def run_nmap(target, args=['-sS','-p-','-T4']):`  
    `with tempfile.NamedTemporaryFile(delete=False, suffix='.xml') as tmp:`  
        `xml_file = tmp.name`  
    `subprocess.run(['nmap'] + args + ['-oX', xml_file, target], check=True)`  
    `return xml_file`

`def parse_nmap(xml_file):`  
    `with open(xml_file) as f:`  
        `data = xmltodict.parse(f.read())`  
    `hosts = []`  
    `for h in data['nmaprun'].get('host', []):`  
        `addr = None`  
        `if isinstance(h.get('address'), list):`  
            `addr = h['address'][0]['@addr']`  
        `elif isinstance(h.get('address'), dict):`  
            `addr = h['address']['@addr']`  
        `ports = []`  
        `ports_data = h.get('ports', {}).get('port', [])`  
        `if isinstance(ports_data, dict): ports_data = [ports_data]`  
        `for p in ports_data:`  
            `ports.append({`  
                `'port': int(p['@portid']),`  
                `'protocol': p['@protocol'],`  
                `'state': p.get('state', {}).get('@state'),`  
                `'service': p.get('service', {}).get('@name') if p.get('service') else None`  
            `})`  
        `hosts.append({'ip': addr, 'ports': ports})`  
    `return hosts`

`def index_hosts(hosts):`  
    `client = OpenSearch(hosts=[OPENSEARCH])`  
    `idx = 'asm-assets'`  
    `if not client.indices.exists(idx): client.indices.create(index=idx)`  
    `for h in hosts:`  
        `client.index(index=idx, body={'ip': h['ip'], 'ports': h['ports'], 'source': 'nmap'})`

`if __name__ == '__main__':`  
    `import sys`  
    `target = sys.argv[1] if len(sys.argv)>1 else '192.168.56.0/24'`  
    `xml = run_nmap(target)`  
    `hosts = parse_nmap(xml)`  
    `index_hosts(hosts)`  
    `os.remove(xml)`  
    `print('Done.')`

## **Appendix C — Web App (Flask) & Scheduling**

`/opt/asm/webapp/app.py` (simplified):

`from flask import Flask, render_template, request, jsonify`  
`from opensearchpy import OpenSearch`  
`import subprocess, threading`  
`from apscheduler.schedulers.background import BackgroundScheduler`

`app = Flask(__name__)`  
`client = OpenSearch(hosts=[{'host':'localhost','port':9200}])`

`@app.route('/')`  
`def index():`  
    `res = client.search(index='asm-assets', body={"query":{"match_all":{}}}, size=100)`  
    `assets = [h['_source'] for h in res['hits']['hits']]`  
    `return render_template('index.html', assets=assets)`

`@app.route('/vulns')`  
`def vulns():`  
    `res = client.search(index='asm-vulns', body={"query":{"match_all":{}}}, size=100)`  
    `vulns = [h['_source'] for h in res['hits']['hits']]`  
    `return render_template('vulns.html', vulns=vulns)`

`@app.route('/scan', methods=['POST'])`  
`def scan():`  
    `target = request.form.get('target')`  
    `if not target: return jsonify({'status':'error','msg':'no target'}),400`  
    `def job(t): subprocess.run(['python3','/opt/asm/scan_and_index.py', t])`  
    `threading.Thread(target=job, args=(target,)).start()`  
    `return jsonify({'status':'started','target':target})`

`# schedule recurring scans`  
`sched = BackgroundScheduler()`  
`sched.add_job(lambda: subprocess.run(['python3','/opt/asm/scan_and_index.py','10.0.2.0/24']),`  
              `'interval', hours=6)`  
`sched.start()`

## **Appendix D — Nginx Reverse Proxy (HTTPS)**

`server {`  
    `listen 80;`  
    `server_name asm.example.com;`  
    `return 301 https://$host$request_uri;`  
`}`  
`server {`  
    `listen 443 ssl;`  
    `server_name asm.example.com;`  
    `ssl_certificate /etc/letsencrypt/live/asm.example.com/fullchain.pem;`  
    `ssl_certificate_key /etc/letsencrypt/live/asm.example.com/privkey.pem;`

    `location / {`  
        `proxy_pass http://127.0.0.1:8000;  # gunicorn`  
        `proxy_set_header Host $host;`  
        `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`  
    `}`  
`}`

## **Appendix E — Quick Commands (Build Sheet)**

`# base tools`  
`sudo apt update && sudo apt install -y nmap masscan amass python3-venv git docker-compose`

`# python env`  
`python3 -m venv ~/asm-venv && source ~/asm-venv/bin/activate`  
`pip install opensearch-py xmltodict flask apscheduler gunicorn`

`# start OpenSearch stack`  
`cd /opt/asm && docker-compose up -d`

`# smoke test: nmap -> opensearch`  
`python3 /opt/asm/scan_and_index.py 10.0.2.0/24`

`# run webapp (dev)`  
`export FLASK_APP=/opt/asm/webapp/app.py`  
`flask run --host=0.0.0.0 --port=5000`  
