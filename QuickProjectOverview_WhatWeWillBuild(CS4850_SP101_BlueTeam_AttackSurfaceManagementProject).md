# **1\) Quick overview (what we’ll build)**

1. A virtual lab of target assets (Ubuntu web servers, internal services).

2. An ASM host that runs discovery and vulnerability tools (Nmap, Amass, OpenVAS/GVM).

3. An indexing \+ dashboard backend (OpenSearch \+ Dashboards).

4. A small web application (Flask) that shows assets & vulnerabilities, allows on-demand scans, and reads/writes to OpenSearch.

5. Scheduling and automation (cron or APScheduler).

6. Secure, reverse-proxied access to the web UI (Nginx \+ TLS).

---

# **2\) Prerequisites (hardware & base software)**

* Host machine: x86\_64 with 16 GB RAM, ≥ 500 GB disk.

* Software installed on host: `docker`, `docker-compose` (or `podman`/compose), `python3`, `python3-venv`, `git`.

* Create a user for ASM operations (optional): `sudo adduser asm`.

---

# **3\) Lab & networking setup (VMs / containers)**

1. Create isolated networks in VirtualBox or use Docker networks. Suggested topology:

   * `network-external` — place external-facing targets here.

   * `network-internal` — for internal services and ASM to reach them.

2. Create VMs/containers for:

   * ASM host (Ubuntu or Kali) — runs scanning scripts & web app.

   * Target web server(s) (Ubuntu \+ Docker containers running sample web apps).

   * OpenSearch \+ Dashboards (in a Docker stack).

   * Optional: Samba/FreeIPA server to simulate internal services.

You can run everything in containers if you prefer (ASM tools \+ OpenSearch \+ web app in Docker Compose).

---

# **4\) Deploy search/visualization backend (OpenSearch) — Docker Compose**

Use OpenSearch (open-source fork of Elasticsearch) \+ OpenSearch Dashboards. Create `docker-compose.yml`:

`version: '3.8'`  
`services:`  
  `opensearch:`  
    `image: opensearchproject/opensearch:2.10.0`  
    `container_name: opensearch`  
    `environment:`  
      `- discovery.type=single-node`  
      `- "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"`  
      `- plugins.security.disabled=true`  
    `ulimits:`  
      `memlock:`  
        `soft: -1`  
        `hard: -1`  
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
    `depends_on:`  
      `- opensearch`

`volumes:`  
  `opensearch-data:`

Start: `docker-compose up -d`.  
 Confirm `http://localhost:9200` and `http://localhost:5601`.

---

# **5\) Install core ASM tools on the ASM host**

On Ubuntu/Kali, install the basics:

`sudo apt update`  
`sudo apt install -y nmap masscan amass python3-pip python3-venv git`  
`# Optional: OpenVAS/GVM via distro package or Docker image (see next)`

**OpenVAS (GVM)**: install via your distro package if available, or run a GVM Docker image for simpler lab setup (search for trusted GVM community images). Once installed, schedule periodic scans and export reports in XML for indexing.

---

# **6\) Create automation scripts — discovery → index**

Goal: run scanners, parse results, push to OpenSearch indexes.

Install Python deps on ASM host:

`python3 -m venv ~/asm-venv`  
`source ~/asm-venv/bin/activate`  
`pip install opensearch-py xmltodict flask apscheduler gunicorn`

Example `scan_and_index.py` (Nmap → OpenSearch). Save to `/opt/asm/scan_and_index.py`:

`#!/usr/bin/env python3`  
`import subprocess, tempfile, xmltodict, json, os`  
`from opensearchpy import OpenSearch`

`OPENSEARCH = {'host':'localhost','port':9200}   # change host if opensearch is remote`

`def run_nmap(target, args=['-sS','-p-','-T4']):`  
    `with tempfile.NamedTemporaryFile(delete=False, suffix='.xml') as tmp:`  
        `xml_file = tmp.name`  
    `cmd = ['nmap'] + args + ['-oX', xml_file, target]`  
    `subprocess.run(cmd, check=True)`  
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
        `# services`  
        `ports = []`  
        `ports_data = h.get('ports', {}).get('port', [])`  
        `if isinstance(ports_data, dict):`  
            `ports_data = [ports_data]`  
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
    `# create index if missing`  
    `if not client.indices.exists(idx):`  
        `client.indices.create(index=idx)`  
    `for h in hosts:`  
        `doc = {`  
            `'ip': h['ip'],`  
            `'ports': h['ports'],`  
            `'source': 'nmap'`  
        `}`  
        `client.index(index=idx, body=doc)`

`if __name__ == '__main__':`  
    `import sys`  
    `target = sys.argv[1] if len(sys.argv)>1 else '192.168.56.0/24'`  
    `xml = run_nmap(target)`  
    `hosts = parse_nmap(xml)`  
    `index_hosts(hosts)`  
    `os.remove(xml)`  
    `print('Done.')`

Run manually: `python scan_and_index.py 10.0.2.15` (or a CIDR). This will create `asm-assets` index in OpenSearch.

**Notes:** Add similar parsers for Amass, Masscan and GVM/OpenVAS XML export. Index vulnerabilities into `asm-vulns` with fields like `cvss`, `cve`, `description`, `host_ip`, `service`, `first_seen`.

---

# **7\) Build the web interface (Flask) — a simple dashboard & control panel**

Create `/opt/asm/webapp/app.py`:

`from flask import Flask, render_template, request, jsonify`  
`from opensearchpy import OpenSearch`  
`import subprocess, threading`

`app = Flask(__name__)`  
`client = OpenSearch(hosts=[{'host':'localhost','port':9200}])`

`@app.route('/')`  
`def index():`  
    `res = client.search(index='asm-assets', body={"query":{"match_all":{}}}, size=100)`  
    `assets = [hit['_source'] for hit in res['hits']['hits']]`  
    `return render_template('index.html', assets=assets)`

`@app.route('/vulns')`  
`def vulns():`  
    `res = client.search(index='asm-vulns', body={"query":{"match_all":{}}}, size=100)`  
    `vulns = [hit['_source'] for hit in res['hits']['hits']]`  
    `return render_template('vulns.html', vulns=vulns)`

`@app.route('/scan', methods=['POST'])`  
`def scan():`  
    `target = request.form.get('target')`  
    `if not target:`  
        `return jsonify({'status':'error','msg':'no target'}),400`  
    `# run scan async to keep web UI responsive`  
    `def job(t):`  
        `subprocess.run(['python3','/opt/asm/scan_and_index.py', t])`  
    `threading.Thread(target=job, args=(target,)).start()`  
    `return jsonify({'status':'started','target':target})`

Create simple templates `templates/index.html` and `templates/vulns.html` using Bootstrap to show tables and a scan form. Example `index.html`:

`<!doctype html>`  
`<html>`  
`<head><title>ASM Dashboard</title>`  
`<link href="https://cdn.jsdelivr.net/npm/bootstrap@5/dist/css/bootstrap.min.css" rel="stylesheet"></head>`  
`<body class="p-4">`  
  `<h1>Assets</h1>`  
  `<form method="post" action="/scan" id="scanForm">`  
    `<input name="target" placeholder="target or CIDR" />`  
    `<button type="submit">Start Scan</button>`  
  `</form>`  
  `<table class="table">`  
    `<thead><tr><th>IP</th><th>Ports</th></tr></thead>`  
    `<tbody>`  
      `{% for a in assets %}`  
      `<tr>`  
        `<td>{{ a.ip }}</td>`  
        `<td>`  
          `{% for p in a.ports %} {{p.port}}/{{p.protocol}} ({{p.state}}) <br/> {% endfor %}`  
        `</td>`  
      `</tr>`  
      `{% endfor %}`  
    `</tbody>`  
  `</table>`  
`<script>`  
`document.getElementById('scanForm').addEventListener('submit', async (e)=>{`  
  `e.preventDefault();`  
  `let form = new FormData(e.target);`  
  `let resp = await fetch('/scan',{method:'POST',body:form});`  
  `let r=await resp.json();`  
  `alert(JSON.stringify(r));`  
`});`  
`</script>`  
`</body>`  
`</html>`

Run locally for testing:

`# inside venv`  
`export FLASK_APP=/opt/asm/webapp/app.py`  
`flask run --host=0.0.0.0 --port=5000`

For production use: run with Gunicorn behind Nginx.

---

# **8\) Schedule scans & automation**

* Use `cron` or `APScheduler` inside the Flask app to run scheduled scans every N hours. Example using APScheduler:

`from apscheduler.schedulers.background import BackgroundScheduler`  
`sched = BackgroundScheduler()`  
`sched.add_job(lambda: subprocess.run(['python3','/opt/asm/scan_and_index.py','10.0.2.0/24']), 'interval', hours=6)`  
`sched.start()`

* For heavier tasks, use a job queue (Celery \+ Redis) — optional.

---

# **9\) Ingest OpenVAS/GVM reports**

1. Configure GVM to run authenticated scans against targets.

2. Export GVM reports as XML.

3. Write a parser (similar to Nmap parser) that extracts vulnerabilities, CVEs, CVSS scores and indexes them to `asm-vulns`.

4. Link `asm-vulns` docs to `asm-assets` documents by IP or asset ID for easy display in web UI.

---

# **10\) Make the website accessible & secure**

1. Reverse proxy with Nginx:

   * Proxy `http://localhost:5000` to `https://asm.yourlab.local`.

   * Use `auth_basic` or Flask-Login for access control.

2. Add TLS:

   * For public access: use Let's Encrypt Certbot with your domain.

   * For lab-only: self-signed cert or internal CA.

3. Harden:

   * Run Gunicorn with a non-root user.

   * Use UFW to restrict ports (allow only 22, 443, 5601 if you want to allow OpenSearch Dashboards).

   * Secure OpenSearch with authentication (if multi-user) or run behind VPN.

Example minimal Nginx server block:

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
        `proxy_pass http://127.0.0.1:8000;   # gunicorn`  
        `proxy_set_header Host $host;`  
        `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`  
    `}`  
`}`

---

# **11\) Visualization & reporting**

* Quick option: use OpenSearch Dashboards (Kibana alternative) to create visual dashboards for counts of discovered assets, top CVSS, frequent ports, trend over time.

* Web UI: add pages for asset search, vulnerability detail, remediation suggestions.

* Export reports: generate PDF/CSV using report tools or scripts (e.g., `weasyprint` or `wkhtmltopdf`).

---

# **12\) Testing & evaluation (how to demonstrate for your capstone)**

1. **Discovery accuracy**: seed your lab with known assets and confirm they’re listed.

2. **Timeliness**: schedule a change (spin up new container) and measure time until detection.

3. **Vulnerability detection**: intentionally deploy a vulnerable web app (e.g., an old version) and show OpenVAS/Nikto detects it and indexed in `asm-vulns`.

4. **Remediation pipeline**: pick a high CVSS finding, create a “ticket” (could be a CSV or a placeholder) and show the time to remediate.

5. **Metrics to collect**: number of assets, unique vulnerabilities, average CVSS, mean time to detect (MTTD), number of false positives (manually inspected).

---

# **13\) Backup, logs & retention**

* Configure OpenSearch index lifecycle / snapshots.

* Ship important logs (scan logs, webapp logs) to a dedicated folder and rotate with `logrotate`.

* Back up exported vulnerability reports regularly.

---

# **14\) Security & ethics checklist**

* Only scan assets you own or have written permission to test.

* Never run this on production internet hosts without approval.

* Use lab networks to avoid accidental disruption.

---

# **15\) Deliverables you can show in the capstone demo**

* Live website showing discovered assets & vulnerabilities.

* OpenSearch dashboards showing trends.

* Sample GitHub repo with all scripts (`scan_and_index.py`, Flask app, Docker Compose).

* A short demo script/documentation with steps: run scans, view results, trigger on-demand scan, show remediation workflow.

---

# **16\) Quick commands summary (copy-paste)**

`# base tools`  
`sudo apt update && sudo apt install -y nmap masscan amass python3-venv git docker-compose`

`# create venv and install Python libs`  
`python3 -m venv ~/asm-venv`  
`source ~/asm-venv/bin/activate`  
`pip install opensearch-py xmltodict flask apscheduler gunicorn`

`# start opensearch stack`  
`cd /opt/asm`  
`docker-compose up -d`

`# run a quick nmap -> opensearch`  
`python3 /opt/asm/scan_and_index.py 10.0.2.0/24`

`# run the webapp (dev)`  
`FLASK_APP=/opt/asm/webapp/app.py flask run --host=0.0.0.0 --port=5000`

