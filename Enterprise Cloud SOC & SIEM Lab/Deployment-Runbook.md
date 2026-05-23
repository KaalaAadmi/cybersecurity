# Building a Cloud SOC & SIEM Lab: Threat Detection in Azure

This guide details the step-by-step process for building a fully functional Security Operations Center (SOC) and Security Information and Event Management (SIEM) lab in Azure.

## Project Overview
This project demonstrates the ability to architect cloud infrastructure, deploy centralized logging, configure log shippers, execute common web/network attacks, and analyze the resulting security events. This documentation is structured to accompany a video walkthrough of the security analysis phase.

- **Cloud Provider:** Microsoft Azure
- **SIEM:** Elastic Stack (Elasticsearch & Kibana)
- **Target App:** Damn Vulnerable Web App (DVWA)
- **Log Shipper:** Filebeat
- **Local OS:** macOS

---

## Phase 1: Preparation & SSH Key Generation

Before touching the cloud, generate a secure SSH key pair on your macOS Terminal. This allows secure, passwordless authentication to your virtual machines.

1. Open your Mac **Terminal**.
2. Run the key generation command:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/azure_siem_lab
```
3. Press **Enter** twice to skip the passphrase.
4. Output your public key and copy the entire string to your clipboard:
```bash
cat ~/.ssh/azure_siem_lab.pub
```

## Phase 2: Cloud Infrastructure Setup
Log into [portal.azure.com](https://www.google.com/search?q=https://portal.azure.com&authuser=3) to deploy the network and virtual machines.

### 0. Create Virtual Networks(VNet)
1. Log into `portal.azure.com`.
2. In the top search bar, type **Virtual networks** and click on the service.
3. Click **Create** to start a new VNet.
4. On the **Basics** tab, under **Resource group**, click **Create new** and type `SOC-Lab-RG`.
5. In the **Virtual network name** box, type `SOC-Lab-VNet`.
6. Select your preferred **Region** (pick the one closest to you physically to reduce latency).
7. Click the **Next: IP Addresses** button at the bottom of the screen.
8. Azure will automatically populate an IPv4 address space (usually `10.0.0.0/16`) and a default subnet (usually `10.0.0.0/24`). Leave these defaults exactly as they are; this configuration is perfect for our lab.
9. Click the blue **Review + create** button at the bottom.
10. Wait a few seconds for the "Validation passed" message to appear at the top, then click **Create**.

#### Step 1: Create VNet 1 (Region A)
1. Go to **Virtual networks** and click **Create**.
2. **Resource Group:** `SOC-Lab-RG`
3. **Name:** `VNet-Norway` (or whatever region you choose).
4. **Region:** `(Europe) Norway-East`
5. Leave IP addresses as default and click **Review + Create**, then **Create**.

#### Step 2: Deploy SIEM and Jump Box (Region A)
Since you have a 4 vCPU limit here, these two VMs will max it out (2 vCPUs + 2 vCPUs).
1. Deploy your `SIEM-VM` and `Jump-Box` into the **Norway-East** region.
2. Under the Networking tab, make sure they are both attached to `VNet-NorthEurope`.

#### Step 3: Create VNet 2 (Region B)
1. Go back to **Virtual networks** and click **Create**.
2. **Resource Group:** `SOC-Lab-RG` (Keep everything in the same resource group for easy cleanup later).
3. **Name:** `VNet-Switzerland` (or pick UK South, East US, etc.).
4. **Region:** `(Europe) Switzerland-North`
5. Leave IP addresses as default and click **Create**.

#### Step 4: Deploy DVWA Target (Region B)
1. Deploy your `DVWA-Target` VM.
2. Select **West Europe** as the region.
3. Under the Networking tab, make sure it is attached to `VNet-WestEurope`.
### 1. Create the Virtual Machines
You will create three Ubuntu VMs in a single Resource Group (e.g., `SOC-Lab-RG`). For all VMs, use **Ubuntu Server 22.04 LTS - x64 Gen2** and size **Standard_B2s**.

- **VM 1: Jump-Box** (Your secure gateway)
- **VM 2: SIEM-VM** (Hosts Elasticsearch & Kibana)
- **VM 3: DVWA-Target** (The vulnerable web server)
>Select `(Europe) Norway-East` or `(Europe) Switzerland-North` in the region
- When you are on the "Create a virtual machine" screen, after filling out the "Basics" tab (where you put your SSH keys), click the **Next: Networking** button.
- In the **Virtual network** dropdown, you will now see `SOC-Lab-VNet` as an option. Make sure it is selected. 
- Ensure the **Subnet** dropdown says `default (10.0.0.0/24)`.

_For each VM's Administrator Account settings:_

- Authentication type: **SSH public key**
- Username: `azureuser`
- SSH public key source: **Use existing public key**
- Key data: Paste the public key you copied from your Mac terminal.
### 2. Configure Network Security Groups (NSGs)
By default, Azure blocks inbound traffic. We must explicitly allow necessary ports.

**For the Jump-Box NSG:**
- Add Inbound Rule: Allow Port `22` (SSH) from your Mac's specific public IP address.

**For the SIEM-VM NSG (The Vault):**
Because we are using a Bastion Host, the SIEM must be completely hidden from the public internet.
1. Add Inbound Rule (Management):
   - Source IP: Paste the **Private IP** of the `Jump-Box` VM (e.g., 10.0.0.4).
   - Destination Ports: `22, 5601` (SSH and Kibana)
2. Add Inbound Rule (Log Ingestion):
   - Source IP: Paste the **Public IP** of the `DVWA-Target` VM.
   - Destination Ports: `9200` (Elasticsearch)

**For the DVWA-Target NSG:**
- Add Inbound Rule: Allow Port `80` (HTTP) from `Any`.

### Phase 3: Deploying the Elastic SIEM
We will use Docker to quickly spin up the Elastic Stack.
1. 1. Connect to the SIEM VM by securely hopping through the Jump-Box:
```bash
ssh -i ~/.ssh/azure_siem_lab -A azureuser@<JUMP_BOX_PUBLIC_IP>
```
(Once inside the Jump-Box terminal, connect to the SIEM using its internal IP):
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/azure_siem_lab
ssh -A azureuser@<JUMP_BOX_PUBLIC_IP>
ssh azureuser@10.0.0.5
```
1. Install Docker:
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker
```
3. Create the deployment directory and `docker-compose.yml` file:
```bash
mkdir siem-stack && cd siem-stack
nano docker-compose.yml
```
4. Paste the following configuration (security features disabled for local lab testing):
```yaml
version: '3.7'
services:
  elasticsearch:
	image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
	environment:
	  - discovery.type=single-node
	  - xpack.security.enabled=false
	ports:
	  - "9200:9200"
	networks:
	  - elastic
  kibana:
	image: docker.elastic.co/kibana/kibana:8.12.0
	ports:
	  - "5601:5601"
	environment:
	  - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
	networks:
	  - elastic
	depends_on:
	  - elasticsearch
networks:
  elastic:
	driver: bridge
```
5. Save (`Ctrl+O`, `Enter`), Exit (`Ctrl+X`), and start the stack:
```bash
docker compose up -d
```
6. **Establish a Secure SSH Tunnel:** Because the SIEM is hidden behind the Jump-Box, you must create an encrypted tunnel from your macOS terminal to view the dashboard. Run this command on your Mac:
```bash
ssh -i ~/.ssh/azure_siem_lab -L 5601:<SIEM_VM_PRIVATE_IP>:5601 azureuser@<JUMP_BOX_PUBLIC_IP>
```
   7. Leave that terminal running, open your Mac browser, and navigate to `http://localhost:5601` to verify Kibana is running.
### Phase 4: Deploying the Target App (DVWA)
1. Open a new Mac terminal tab and connect to the DVWA VM:
```
ssh -i ~/.ssh/azure_siem_lab azureuser@<DVWA_TARGET_PUBLIC_IP>
```
2. Install Docker using the same commands from Phase 3, step 2.
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker
```
3. **Create a local folder and deploy the container with a Volume Mount.** _(This maps the container's internal logs to the host machine so Filebeat can read them).
```bash
sudo mkdir -p /var/log/apache2
sudo docker run --rm -d -p 80:80 -v /var/log/apache2:/var/log/apache2 vulnerables/web-dvwa
```
4. Verify by visiting `http://<DVWA_TARGET_PUBLIC_IP>` in your browser. (Default credentials: `admin` / `password`).
### Phase 5: Log Shipping Configuration
We will use Filebeat on the DVWA machine to forward system logs to the SIEM.

1. On the DVWA-Target terminal, download and install Filebeat:
```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.0-amd64.deb
sudo dpkg -i filebeat-8.12.0-amd64.deb
```
2. Edit the configuration file:
```bash
sudo vi /etc/filebeat/filebeat.yml
```
3. Update `filebeat.inputs`:
```yaml
filebeat.inputs:
- type: filestream
  id: my-raw-logs
  enabled: true
  paths:
    - /var/log/apache2/access.log
```
4. ~~%%  OMIT %%Update the `setup.kibana` host (uncomment it and add your IP):~~
```yaml
setup.kibana:
  host: "http://localhost:5601"
```
4. Update the `output.elasticsearch` hosts to point to the SIEM's database port: 
```yaml 
output.elasticsearch:
  hosts: ["http://<SIEM_VM_PUBLIC_IP>:9200"]
  pipeline: "decode_web_logs"
```
do **not** run `sudo service filebeat start` until you have completed Step 1 of Phase 6 below.
5. Wipe the registry (to ensure it reads from the beginning) and start the service:
```bash
sudo rm -rf /var/lib/filebeat/registry
sudo service filebeat start
```
6. Verify the service is running successfully:
```bash
sudo service filebeat status
```
(You should see `active (running)`. Press `q` to exit the status screen).

>If a recruiter or technical interviewer asks why you didn't just use `sudo filebeat modules enable apache`, you now have a fantastic answer.

>You can confidently say: _"I initially tried the default modules, but the pre-built ingest pipelines were failing to parse the Docker-mounted logs and causing the Filebeat daemon to crash. To ensure reliable log delivery, I disabled the fragile modules, engineered a custom raw `filestream` input, and handled the querying manually in Kibana."_ That is music to a hiring manager's ears!

### **Phase 6: Threat Emulation & Detection**
This phase proves the lab is functional by executing a real-world web application attack and tracking the resulting telemetry through the SIEM.
**Step 1: Create the Ingest Pipeline (Kibana)** 
Before starting the log shipper, configure Elasticsearch to decode URL-encoded web traffic automatically.
1. Open your browser and navigate to Kibana (`http://localhost:5601`).
2. Open the left menu, scroll down to **Management**, and click **Dev Tools**.
3. In the console, paste the following API call to create the pipeline:
```json
PUT _ingest/pipeline/decode_web_logs
{
  "description": "Decode URL-encoded payloads in raw log messages",
  "processors": [
	{
	  "urldecode": {
		"field": "message",
		"ignore_missing": true,
		"ignore_failure": true
	  }
	}
  ]
}
```
OR
```json
PUT _ingest/pipeline/decode_web_logs
{
  "description": "Parse Apache logs and decode URL payloads",
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": [
          "%{IPORHOST:source.ip} %{USER:user.id} %{USER:user.name} \\[%{HTTPDATE:@timestamp}\\] \"%{WORD:http.request.method} %{DATA:url.original} HTTP/%{NUMBER:http.version}\" %{NUMBER:http.response.status_code} %{NUMBER:http.response.bytes} \"%{DATA:http.request.referrer}\" \"%{DATA:user_agent.original}\""
        ],
        "ignore_missing": true,
        "ignore_failure": true
      }
    },
    {
      "urldecode": {
        "field": "url.original",
        "ignore_missing": true,
        "ignore_failure": true
      }
    }
  ]
}
```
4. Click the **Play button** (▶) to execute. You should see `{"acknowledged": true}` on the right screen.
5. _Now you can safely run `sudo service filebeat start` on your DVWA machine._
### Phase 7: Attack Execution & Detection
#### SQL Injection Attack:
##### Step 2: Trigger the SQL Injection Attack
Now, let's generate the malicious traffic.
1. Open a web browser on your Mac and go to `http://<DVWA_TARGET_PUBLIC_IP>`.
2. Log in (Username: `admin`, Password: `password`).
3. On the left-hand menu, look for **DVWA Security**. Click it, change the security level dropdown from "Impossible" or "Medium" to **low**, and hit **Submit**. _(This ensures the attack goes through successfully)._
4. Now, click **SQL Injection** on the left menu.
5. In the User ID box, type exactly this:
```
%' or '1'='1
```
6. Click **Submit**. You will see the web page reload and output a list of users. The attack is complete.
##### Step 3: Hunt for the Attack in Kibana (Step-by-Step Clicks)
Now, let's switch hats to a SOC Analyst and find this attack in the SIEM dashboard.
1. Open a new tab in your browser and open Kibana: `http://localhost:5601`.
2. Look at the top-left corner of the screen and click the **Hamburger Menu icon** (three horizontal lines).
3. Under the **Analytics** section, click on **Discover**.
4. In the top-left area of the main screen, ensure your Data View dropdown is set to `filebeat-*`.
5. Look at the top-right corner of the screen and click the **Time Filter** (it might say "Last 15 minutes"). Change it to **Last 1 hour** to make sure you capture your recent activity.
###### The KQL Queries to Use
At the top of the screen, you will see a long search bar. This is where you write **KQL (Kibana Query Language)**.
To isolate the web server logs and find the attack, copy and paste this exact query into the search bar and press **Enter**:
```
"1'='1"
```

> **What this query does:** Because we built a custom Elasticsearch Ingest Pipeline to automatically decode URL characters (like `%25%27`), we can search for the exact clear-text SQL injection payload across all our log data, simulating how a SOC analyst hunts for specific Threat Intelligence indicators.

##### Step 4: Analyzing the Log Details (What to Screenshot)
Once you hit enter, you should see a list of results appear in the table below the graph.
1. Click on the small **`>` (expand) arrow** next to the very top log event in the list. This expands the log to show all details.
2. Switch the view from **Table** to **JSON** (there are tabs right at the top of the expanded log).
3. Scroll through the fields to find the following key items to screenshot for your portfolio:
	- **`url.original` or `url.query`:** You will see something like `/vulnerabilities/sqli/?id=%25%27+or+%271%27%3D%271&Submit=Submit`. Notice how the spaces became `+` and characters like `'` became `%27`. **Screenshot this field**—it proves you can identify raw URL encoding of an attack.
	- **`source.ip`:** This shows the public IP address of the attacker (your home network's IP). **Screenshot this** to demonstrate how an analyst identifies the origin of a threat.
	- **`http.response.status_code`:** It will likely show `200`. This means the server successfully processed the request, indicating to a SOC analyst that the exploit attempt likely succeeded and needs immediate investigation.
