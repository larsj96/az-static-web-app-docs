## Hardware Specifications

### Servers:
- **ProLiant DL380 Gen9 (x2)**
  - CPUs: 2x Intel(R) Xeon(R) CPU E5-2667 v4 @ 3.20GHz
  - RAM: 768 GB
  - Storage: 4 TB NVME each

- **ProLiant DL380 Gen9 (x1)**
  - CPUs: 2x Intel(R) Xeon(R) CPU E5-2687W v3 @ 3.10GHz
  - RAM: 384 GB
  - Storage: 3 TB NVME

- **Dell R820 (x1) - Server not in use anymore**
  - CPUs: 4x Intel(R) Xeon(R) CPU E5-4640 0 @ 2.40GHz
  - RAM: 640 GB DDR3
  - Storage: Not in use anymore

### Networking Equipment:
- **Firewall and Upstream Internet:**
  - Fortigate 500D that are managed with terraform and gitlab pipelines

- **Access Point:**
  - Ubiquiti UniFi UAP-AC-PRO
  - VLANs/SSID: one for guest/IOT and other for mgmt

- **Switches:**
  - Zyxel Switch XGS1250-12
    - Configuration: 10 Gbit network for CEPH storage for the Proxmox nodes
  - MikroTik Cloud Router Switch CRS326-24G-2S+RM
    - Usage: VM internet traffic / Proxmox MGMT

### Other Equipment:
- **Cooling:**
  - Cloudline T6 exhaust fan from the server rack to my office

  ![Server room  ](serverroom.jpg "Homemade rack :)")
