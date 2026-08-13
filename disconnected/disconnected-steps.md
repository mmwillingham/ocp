# OpenShift 4.20 Disconnected Mirroring & OSUS Installation Runbook
## Section 1 - AWS Bastion Configuration (Run on Local Laptop)
```
# Configure AWS CLI Credentials
aws configure

# Lookup Security Group ID and allow required ports
export BASTION_NAME="bastion"
export INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=*${BASTION_NAME}*" "Name=instance-state-name,Values=running,stopped" --query "Reservations[0].Instances[0].InstanceId" --output text)
export SG_ID=$(aws ec2 describe-instances --instance-ids ${INSTANCE_ID} --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)

# Allow Nexus UI (8081/8443) and Docker Registry Ports (5001-5003)
aws ec2 authorize-security-group-ingress --group-id ${SG_ID} --protocol tcp --port 8081 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SG_ID} --protocol tcp --port 8443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SG_ID} --protocol tcp --port 5001-5003 --cidr 0.0.0.0/0

# Stop Instance, Upgrade to m5.2xlarge, and Restart
aws ec2 stop-instances --instance-ids ${INSTANCE_ID}
aws ec2 wait instance-stopped --instance-ids ${INSTANCE_ID}
aws ec2 modify-instance-attribute --instance-id ${INSTANCE_ID} --instance-type '{"Value": "m5.2xlarge"}'
aws ec2 start-instances --instance-ids ${INSTANCE_ID}

# Wait for Instance OS Initialization
while [ "$(aws ec2 describe-instance-status --instance-ids ${INSTANCE_ID} --query "InstanceStatuses[0].InstanceStatus.Status" --output text)" != "ok" ]; do
  echo "Waiting for instance OS initialization..."; sleep 5;
done

# Expand Root EBS Volume to 800 GB
export VOLUME_ID=$(aws ec2 describe-instances --instance-ids ${INSTANCE_ID} --query "Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId" --output text)
aws ec2 modify-volume --volume-id ${VOLUME_ID} --size 800

# Wait for Volume Modification
while [ "$(aws ec2 describe-volumes-modifications --volume-id ${VOLUME_ID} --query "VolumesModifications[0].ModificationState" --output text)" = "modifying" ]; do
  echo "Waiting for EBS volume modification..."; sleep 5;
done
```

## Section 2: Bastion Filesystem & Workspace Preparation (Run on Bastion)
```
# Expand Filesystem
export ROOT_DISK=$(lsblk -no PKNAME $(findmnt -n -o SOURCE /))
export ROOT_PART_NUM=$(lsblk -no KNAME $(findmnt -n -o SOURCE /) | grep -o '[0-9]*$')

sudo growpart /dev/${ROOT_DISK} ${ROOT_PART_NUM} || true
sudo xfs_growfs /
df -h /

# Clean Old Workspaces & Containers
rm -rf ~/oc-mirror-workspace ~/cincinnati-graph-data ~/all
podman stop -a --ignore
podman rm -a --force --ignore
podman system prune -a --volumes --force
sudo rm -rf /var/nexus-data /etc/nexus-ssl /etc/nginx
sudo podman system prune -a --volumes --force
```

## Section 3: Install & Configure Nexus Registry (Run on Bastion)
```
# Create Storage and SSL Directories
sudo mkdir -p /var/nexus-data /etc/nexus-ssl /etc/nginx
sudo chown -R 200:200 /var/nexus-data /etc/nexus-ssl

# Get Public IP
export BASTION_IP=$(curl -s https://ifconfig.me || hostname -I | awk '{print $1}')

# Generate Self-Signed Certificate
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nexus-ssl/nexus.key \
  -out /etc/nexus-ssl/nexus.crt \
  -subj "/CN=${BASTION_IP}" \
  -addext "subjectAltName=IP:${BASTION_IP},IP:127.0.0.1,DNS:localhost"

sudo chmod 644 /etc/nexus-ssl/nexus.crt /etc/nexus-ssl/nexus.key

# Allow Firewall Traffic & Launch Nexus Container with Auto-Restart
sudo iptables -I INPUT 1 -p tcp -m multiport --dports 8081,8443,5001,5002 -j ACCEPT

sudo podman run -d --name nexus --replace \
  --restart=always \
  -p 8081:8081 \
  -e INSTALL4J_ADD_VM_PARAMS="-Xms512m -Xmx1024m -XX:MaxDirectMemorySize=512m" \
  -v /var/nexus-data:/nexus-data:Z \
  docker.io/sonatype/nexus3:latest

# Wait for Nexus Startup
while [ "$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8081/)" != "200" ]; do
  echo "Waiting for Nexus 3 initialization..."; sleep 5;
done

# Print Admin Password
echo "Initial Nexus Admin Password:"
sudo podman exec nexus cat /nexus-data/admin.password && echo ""
echo "Nexus URL: http://${BASTION_IP}:8081"
```

### Manual Nexus steps in Browser
```
Open http://<BASTION_IP>:8081 > Login as admin with the password printed above
Change admin password to: RedHat123!
Login again and refresh browser to complete wizard.
Settings > Security > Anonymous Access > Check "Allow anonymous users..." > Save
Settings > Security > Realms > Move "Docker Bearer Token Realm" to Active > Save
Settings > Repository > Create Repository:
  Format: docker (proxy)
  Name: redhat-proxy
  Other Connectors: HTTPS > 5001
  Allow anonymous pull: Checked
  Remote storage: https://registry.redhat.io
  Authentication: Red Hat Service Account token (from access.redhat.com)
Settings > Repository > Create Repository:Format: docker (hosted)
  Name: ocp-hosted
  Other Connectors: HTTPS > 5002
  Allow anonymous pull: Checked
  Deployment policy: Allow redeploy
```
```
# Configure Nginx SSL Reverse Proxy
cat << 'EOF' | sudo tee /etc/nginx/nexus-proxy.conf
events {
    worker_connections 1024;
}
http {
    client_max_body_size 0;
    chunked_transfer_encoding on;
    ssl_certificate /etc/nexus-ssl/nexus.crt;
    ssl_certificate_key /etc/nexus-ssl/nexus.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    server {
        listen 5001 ssl;
        server_name _;
        location / {
            proxy_pass http://localhost:8081/repository/redhat-proxy/;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header X-Forwarded-Port 5001;
        }
    }

    server {
        listen 5002 ssl;
        server_name _;
        location / {
            proxy_pass http://localhost:8081/repository/ocp-hosted/;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header X-Forwarded-Port 5002;
        }
    }
}
EOF

# Launch Nginx SSL Proxy Container
sudo podman run -d --name nexus-ssl-proxy --replace \
  --net=host \
  --restart=always \
  -v /etc/nginx/nexus-proxy.conf:/etc/nginx/nginx.conf:ro \
  -v /etc/nexus-ssl:/etc/nexus-ssl:ro \
  docker.io/library/nginx:alpine

# Trust Certificate on Bastion Host
sudo cp /etc/nexus-ssl/nexus.crt /etc/pki/ca-trust/source/anchors/nexus.crt
sudo update-ca-trust

# Test Ports
curl -I https://localhost:5001/v2/
curl -I https://localhost:5002/v2/

# Resume Check (If Bastion Instance Was Restarted)
sudo podman start nexus nexus-ssl-proxy 2>/dev/null || true
until [ "$(curl -k -s -o /dev/null -w "%{http_code}" https://localhost:5002/v2/)" != "502" ]; do
  echo "Waiting for Nexus initialization..."; sleep 5;
done
echo "✅ Nexus is online and responding!"
```

## Section 4: Install oc-mirror v2 & Mirror Content (Run on Bastion)
```
# Install oc-mirror CLI
curl -sL https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/oc-mirror.tar.gz | tar -xz -C /tmp
sudo mv /tmp/oc-mirror /usr/local/bin/oc-mirror
sudo chmod +x /usr/local/bin/oc-mirror

WORKSPACE="${HOME}/oc-mirror-workspace"
mkdir -p "${WORKSPACE}"
oc-mirror version

# Write ImageSetConfiguration
cat << 'EOF' > "${WORKSPACE}/imageset-config.yaml"
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
mirror:
  platform:
    channels:
    - name: stable-4.20
      minVersion: 4.20.26
      maxVersion: 4.20.26
    graph: true
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
      packages:
        - name: advanced-cluster-management
          defaultChannel: release-2.17
          channels:
            - name: release-2.17
              minVersion: '2.17.0'
              maxVersion: '2.17.0'
        - name: cincinnati-operator
          defaultChannel: v1
          channels:
            - name: v1
              minVersion: '5.0.3'
              maxVersion: '5.0.3'
        - name: cluster-kube-descheduler-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '5.3.2'
              maxVersion: '5.3.2'
        - name: cluster-logging
          defaultChannel: stable-6.6
          channels:
            - name: stable-6.6
              minVersion: '6.6.0'
              maxVersion: '6.6.0'
        - name: cluster-observability-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '1.5.1'
              maxVersion: '1.5.1'
        - name: compliance-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '1.9.1'
              maxVersion: '1.9.1'
        - name: fence-agents-remediation
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '0.6.1'
              maxVersion: '0.6.1'
        - name: file-integrity-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '1.4.0'
              maxVersion: '1.4.0'
        - name: kubernetes-nmstate-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '4.20.0-202608050632'
              maxVersion: '4.20.0-202608050632'
        - name: kubevirt-hyperconverged
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '4.20.24'
              maxVersion: '4.20.24'
        - name: loki-operator
          defaultChannel: stable-6.6
          channels:
            - name: stable-6.6
              minVersion: '6.6.0'
              maxVersion: '6.6.0'
        - name: metallb-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '4.20.0-202608040713'
              maxVersion: '4.20.0-202608040713'
        - name: mtv-operator
          defaultChannel: release-v2.12
          channels:
            - name: release-v2.12
              minVersion: '2.12.5'
              maxVersion: '2.12.5'
        - name: netobserv-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '1.12.1'
              maxVersion: '1.12.1'
        - name: nfd
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '4.20.0-202608040713'
              maxVersion: '4.20.0-202608040713'
        - name: node-healthcheck-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '0.10.3'
              maxVersion: '0.10.3'
        - name: node-maintenance-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '5.5.0'
              maxVersion: '5.5.0'
        - name: node-observability-operator
          defaultChannel: alpha
          channels:
            - name: alpha
              minVersion: '0.2.0'
              maxVersion: '0.2.0'
        - name: numaresources-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '4.20.3'
              maxVersion: '4.20.3'
        - name: openshift-gitops-operator
          defaultChannel: latest
          channels:
            - name: latest
              minVersion: '1.21.2'
              maxVersion: '1.21.2'
        - name: redhat-oadp-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '1.5.7'
              maxVersion: '1.5.7'
    - catalog: registry.redhat.io/redhat/certified-operator-index:v4.20
      packages:
        - name: dynatrace-operator
          defaultChannel: alpha
          channels:
            - name: alpha
              minVersion: '1.10.2'
              maxVersion: '1.10.2'
        - name: infinibox-operator-certified
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '2.29.0'
              maxVersion: '2.29.0'
        - name: vault-secrets-operator
          defaultChannel: stable
          channels:
            - name: stable
              minVersion: '1.5.0'
              maxVersion: '1.5.0'
  additionalImages: []
EOF

# Sanitize non-breaking spaces
sed -i 's/\xc2\xa0/ /g' "${WORKSPACE}/imageset-config.yaml"

# Configure Merged Credentials File
NEXUS_USER="admin"
NEXUS_PASS="RedHat123!"
NEXUS_AUTH=$(echo -n "${NEXUS_USER}:${NEXUS_PASS}" | tr -d '\r\n' | base64 | tr -d '\r\n')

mkdir -p ~/.open-shift
jq --arg auth "$NEXUS_AUTH" '.auths["localhost:5002"] = {"auth": $auth}' ~/pull-secret.json > ~/.open-shift/containers-auth.json

# Run Mirroring Process
oc-mirror --v2 \
  --config "${WORKSPACE}/imageset-config.yaml" \
  docker://localhost:5002 \
  --authfile ~/.open-shift/containers-auth.json \
  --workspace "file://${WORKSPACE}" \
  --remove-signatures
```

## Section 5: Prepare Cluster Manifests & Apply SSL Trust First (Run on Bastion)
```
RESOURCE_DIR=$(find "${HOME}" -type d -name "cluster-resources" | head -n 1)
echo "Found cluster resources at: ${RESOURCE_DIR}"

# 1. Cleanly inject mirrorSourcePolicy: NeverContactSource
for f in "${RESOURCE_DIR}"/idms*.yaml "${RESOURCE_DIR}"/itms*.yaml; do
  [ -f "$f" ] || continue
  sed -i '/mirrorSourcePolicy:/d' "$f"
  sed -i 's/^\([[:space:]]*\)source:/\1mirrorSourcePolicy: NeverContactSource\n\1source:/' "$f"
done

# 2. Replace localhost:5002 with BASTION_HOST:5002
BASTION_HOST=$(curl -s --connect-timeout 2 http://169.254.169.254/latest/meta-data/public-ipv4 2>/dev/null \
  || ip route get 1.1.1.1 2>/dev/null | grep -oP 'src \K\S+' \
  || hostname -I | awk '{print $1}')

echo "Using Bastion Host IP: ${BASTION_HOST}"

sed -i "s/localhost:5002/${BASTION_HOST}:5002/g" "${RESOURCE_DIR}"/*.yaml
sed -i "s/localhost:5002/${BASTION_HOST}:5002/g" "${RESOURCE_DIR}"/*.json 2>/dev/null || true

# 3. Sanity check: verify no remaining localhost endpoints
if grep -rn "localhost:5002" "${RESOURCE_DIR}"/*.yaml "${RESOURCE_DIR}"/*.json 2>/dev/null; then
  echo "🛑 FAIL: Found un-replaced localhost endpoints in manifests above!"
  exit 1
else
  echo "✅ PASS: All manifests cleanly point to ${BASTION_HOST}:5002"
fi

# 4. Configure Cluster Registry CA Trust FIRST (Crucial Order!)
REGISTRY_KEY="${BASTION_HOST}..5002"

oc create configmap registry-cas -n openshift-config \
  --from-file=updateservice-registry=/etc/nexus-ssl/nexus.crt \
  --from-file="${REGISTRY_KEY}"=/etc/nexus-ssl/nexus.crt \
  --dry-run=client -o yaml | oc apply -f -

cat <<EOF | oc apply -f -
apiVersion: config.openshift.io/v1
kind: Image
metadata:
  name: cluster
spec:
  additionalTrustedCA:
    name: registry-cas
EOF
```

## Section 6: Apply IDMS, ITMS, Signatures & Wait for Node Updates (Run on Bastion)
```
# Apply IDMS, ITMS, and Signature ConfigMap
oc apply -f "${RESOURCE_DIR}/idms-oc-mirror.yaml"
oc apply -f "${RESOURCE_DIR}/itms-oc-mirror.yaml"
oc apply -f "${RESOURCE_DIR}/signature-configmap.yaml"

# BLOCKING WAIT: MachineConfigPool Node Updates
echo "⏳ Waiting for MachineConfigPools to begin and complete updates..."
sleep 15

until [ "$(oc get mcp -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Updating")].status}{"\n"}{end}' | grep -c "True")" -eq 0 ] && \
      [ "$(oc get mcp -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Updated")].status}{"\n"}{end}' | grep -v "True" | wc -l)" -eq 0 ]; do
  echo "[$(date +'%H:%M:%S')] Nodes applying registry configuration updates..."
  sleep 20
done
echo "✅ ALL NODES & MACHINECONFIGPOOLS ARE FULLY UPDATED AND READY!"
```

## Section 7: Apply CatalogSources & Disable Defaults (Run on Bastion)
```
# Apply CatalogSources and ClusterCatalogs
find "${RESOURCE_DIR}" -type f \( -name "cs-*.yaml" -o -name "cc-*.yaml" \) -exec oc apply -f {} \;

# Disable Default Public OperatorHub Catalog Sources
cat << EOF | oc apply -f -
apiVersion: config.openshift.io/v1
kind: OperatorHub
metadata:
  name: cluster
spec:
  disableAllDefaultSources: true
EOF

# BLOCKING WAIT: CatalogSource Readiness
CATALOG_NAME="cs-redhat-operator-index-v4-20"

echo "⏳ Waiting for CatalogSource ${CATALOG_NAME} to reach READY state..."
until [ "$(oc get catalogsource ${CATALOG_NAME} -n openshift-marketplace -o jsonpath='{.status.connectionState.lastObservedState}' 2>/dev/null)" = "READY" ]; do
  STATUS=$(oc get catalogsource ${CATALOG_NAME} -n openshift-marketplace -o jsonpath='{.status.connectionState.lastObservedState}' 2>/dev/null)
  echo "[$(date +'%H:%M:%S')] CatalogSource status: ${STATUS:-Pending}"
  sleep 10
done
echo "✅ CATALOGSOURCE IS READY AND CONNECTED!"
oc get catalogsource -n openshift-marketplace
```

## Section 8: Deploy OpenShift Update Service (OSUS) & Patch CVO (Run on Bastion)
```
NS="openshift-update-service"

# 1. Create OSUS Namespace, OperatorGroup, and Subscription
cat << 'EOF' | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-update-service
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-update-service-og
  namespace: openshift-update-service
spec:
  targetNamespaces:
  - openshift-update-service
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cincinnati-operator
  namespace: openshift-update-service
spec:
  channel: v1
  name: cincinnati-operator
  source: cs-redhat-operator-index-v4-20
  sourceNamespace: openshift-marketplace
EOF

# 2. BLOCKING WAIT: Cincinnati Operator CSV Installation
echo "⏳ Waiting for Cincinnati Operator installation..."
until oc get csv -n "${NS}" 2>/dev/null | grep -E -i "update-service|cincinnati" | grep -i "Succeeded" >/dev/null 2>&1; do
  echo "[$(date +'%H:%M:%S')] Operator installation in progress..."
  sleep 10
done
echo "✅ CINCINNATI OPERATOR INSTALLED SUCCESSFULLY!"

# 3. Apply UpdateService Custom Resource NOW THAT CRD EXISTS
RESOURCE_DIR=$(find "${HOME}" -type d -name "cluster-resources" | head -n 1)
oc apply -f "${RESOURCE_DIR}/updateService.yaml" -n "${NS}"

# 4. BLOCKING WAIT: Operator Reconciliation & Deployment Rollout
echo "⏳ Waiting for operator to reconcile and create deployment object..."
until oc get deployment update-service-oc-mirror -n "${NS}" >/dev/null 2>&1; do
  echo "[$(date +'%H:%M:%S')] Operator reconciling... waiting for deployment object..."
  sleep 5
done

echo "⏳ Deployment object created! Tracking pod rollout..."
oc rollout status deployment/update-service-oc-mirror -n "${NS}" --timeout=300s
echo "✅ UPDATE SERVICE DEPLOYMENT IS 100% READY!"

# 5. Fetch Policy Engine Route & Patch Cluster Version Operator
POLICY_ENGINE_URL=$(oc get route -n "${NS}" -l app=update-service-oc-mirror -o jsonpath='{.items[0].spec.host}')

echo "Patching CVO with Upstream URL: https://${POLICY_ENGINE_URL}/api/upgrades_info/v1/graph"

oc patch clusterversion version --type=json \
  -p '[{"op": "add", "path": "/spec/upstream", "value": "https://'$POLICY_ENGINE_URL'/api/upgrades_info/v1/graph"}]'

# 6. Final Verification
echo -e "\n=================================================="
echo "🎯 FINAL VERIFICATION"
echo "=================================================="
echo "CVO Upstream URL: $(oc get clusterversion -o jsonpath='{.items[*].spec.upstream}')"
echo -e "\nPod Status in ${NS}:"
oc get pods -n "${NS}"
```

## Section 9: OSUS Usage Guide in OpenShift Web Console
```
Once OSUS is deployed and CVO is patched with your local upstream route, OpenShift automatically integrates your mirrored graph directly into the Web Console UI.
- Log in to the OpenShift Web Console as cluster-admin.
- Navigate to Administration > Cluster Settings > Details tab.
- Verify that the Update status section reflects releases pulled from your local route URL:
    https://update-service-oc-mirror-route-openshift-update-service.../api/upgrades_info/v1/graph
- Click Select version (or Update) to choose your target release version from the local graph and initiate the cluster upgrade.
```

## Section 10: Air-Gap Testing & Verification (Optional)
```
# 1. CoreDNS Blackholing inside OpenShift Cluster
oc patch dns.operator.openshift.io/default --type=merge -p '{
  "spec": {
    "zones": [
      {
        "name": "registry.redhat.io",
        "hosts": [{"ip": "127.0.0.1"}]
      },
      {
        "name": "quay.io",
        "hosts": [{"ip": "127.0.0.1"}]
      },
      {
        "name": "cdn.redhat.com",
        "hosts": [{"ip": "127.0.0.1"}]
      },
      {
        "name": "api.openshift.com",
        "hosts": [{"ip": "127.0.0.1"}]
      }
    ]
  }
}'

# 2. Block Local Proxy Feed Port 5001
sudo iptables -I INPUT 1 -p tcp --dport 5001 -j DROP

# 3. Test Bastion Internet Access (Should Succeed)
echo "🌐 Testing Bastion Internet Access..."
curl -s -I https://www.redhat.com | head -n 1

# 4. Test Cluster Isolation (Should Fail/Timeout)
echo "🔒 Testing Cluster Air-Gap Isolation..."
oc debug node/$(oc get nodes -o jsonpath='{.items[0].metadata.name}') -- chroot /host curl -s -I --connect-timeout 3 https://registry.redhat.io/v2/ || echo "✅ SUCCESS: Cluster is fully air-gapped!"

# 5. Check OSUS Upgrade Path
echo "🚀 Checking OSUS Upgrade Status..."
oc adm upgrade
```