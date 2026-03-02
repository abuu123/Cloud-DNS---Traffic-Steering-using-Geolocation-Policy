# Cloud-DNS---Traffic-Steering-using-Geolocation-Policy
Cloud DNS Traffic Steering

#!/bin/bash
# ============================================================
# Cloud DNS Traffic Steering using Geolocation Policy
# ============================================================
# Author: Abu Godo
# Project: Qwiklabs GCP Lab
# ============================================================

# ------------------------------
# 1️⃣ Set active account & project
# ------------------------------
ACTIVE_ACCOUNT="student-01-bd0812d06cce@qwiklabs.net"
gcloud config set account $ACTIVE_ACCOUNT
echo "✅ Active account set to: $ACTIVE_ACCOUNT"

PROJECT_ID=$(gcloud config get-value project)
echo "ℹ️ Using project: $PROJECT_ID"

# ------------------------------
# 2️⃣ Enable required services
# ------------------------------
gcloud services enable compute.googleapis.com dns.googleapis.com
echo "✅ Enabled Compute Engine & Cloud DNS APIs"

# ------------------------------
# 3️⃣ Create firewall rules
# ------------------------------
echo "🔹 Creating firewall rules..."
gcloud compute firewall-rules create fw-default-iapproxy \
  --direction=INGRESS --priority=1000 --network=default \
  --action=ALLOW --rules=tcp:22,icmp --source-ranges=35.235.240.0/20

gcloud compute firewall-rules create allow-http-traffic \
  --direction=INGRESS --priority=1000 --network=default \
  --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 \
  --target-tags=http-server

echo "✅ Firewalls created."

# ------------------------------
# 4️⃣ Create client VMs
# ------------------------------
echo "🔹 Creating client VMs..."
gcloud compute instances create us-client-vm --machine-type=e2-micro --zone=us-east4-b
gcloud compute instances create europe-client-vm --machine-type=e2-micro --zone=europe-central2-c
gcloud compute instances create asia-client-vm --machine-type=e2-micro --zone=asia-east1-a

# ------------------------------
# 5️⃣ Create web VMs with Apache
# ------------------------------
echo "🔹 Creating web VMs with Apache..."
gcloud compute instances create us-web-vm \
  --machine-type=e2-micro --zone=us-east4-b --network=default --subnet=default \
  --tags=http-server \
  --metadata=startup-script='#! /bin/bash
  apt-get update
  apt-get install apache2 -y
  echo "Page served from: us-east4" | tee /var/www/html/index.html
  systemctl restart apache2'

gcloud compute instances create europe-web-vm \
  --machine-type=e2-micro --zone=europe-central2-c --network=default --subnet=default \
  --tags=http-server \
  --metadata=startup-script='#! /bin/bash
  apt-get update
  apt-get install apache2 -y
  echo "Page served from: europe-central2-c" | tee /var/www/html/index.html
  systemctl restart apache2'

# ------------------------------
# 6️⃣ Export web VM IPs
# ------------------------------
US_WEB_IP=$(gcloud compute instances describe us-web-vm --zone=us-east4-b --format="value(networkInterfaces[0].networkIP)")
EUROPE_WEB_IP=$(gcloud compute instances describe europe-web-vm --zone=europe-central2-c --format="value(networkInterfaces[0].networkIP)")

echo "🌐 US Web VM IP: $US_WEB_IP"
echo "🌐 Europe Web VM IP: $EUROPE_WEB_IP"

# ------------------------------
# 7️⃣ Create private DNS zone
# ------------------------------
gcloud dns managed-zones create example \
  --description="Geolocation DNS Test" \
  --dns-name="example.com" \
  --networks=default \
  --visibility=private

echo "✅ Private DNS zone 'example' created."

# ------------------------------
# 8️⃣ Create geolocation-based DNS records
# ------------------------------
gcloud dns record-sets transaction start --zone=example

# Create geolocation items (new recommended format)
gcloud dns record-sets transaction add $US_WEB_IP \
  --name=geo.example.com. --type=A --ttl=300 --zone=example \
  --routing-policy='geo={"locations":["us-east4"]}'

gcloud dns record-sets transaction add $EUROPE_WEB_IP \
  --name=geo.example.com. --type=A --ttl=300 --zone=example \
  --routing-policy='geo={"locations":["europe-central2"]}'

gcloud dns record-sets transaction execute --zone=example

echo "✅ Geolocation DNS records created."

# ------------------------------
# 9️⃣ List DNS records
# ------------------------------
gcloud dns record-sets list --zone=example

echo "🎉 Cloud DNS traffic steering setup complete!"
