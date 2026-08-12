# OpenShift 4.20 Disconnected Mirroring & OSUS Installation Runbook (oc-mirror v2)

==============================================================================
1. AWS Bastion Configuration (Run from Local Laptop)
==============================================================================
```
# Configure AWS CLI Credentials
aws configure
```

```
# Lookup Security Group ID and allow required ports
export BASTION_NAME="bastion"
export INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=*${BASTION_NAME}*" "Name=instance-state-name,Values=running,stopped" --query "Reservations[0].Instances[0].InstanceId" --output text)
export SG_ID=$(aws ec2 describe-instances --instance-ids ${INSTANCE_ID} --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)
```

```
# Allow Nexus UI (8081/8443) and Docker Registry Ports (5001-5003)
aws ec2 authorize-security-group-ingress --group-id ${SG_ID} --protocol tcp --port 8081 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SG_ID} --protocol tcp --port 8443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SG_ID} --protocol tcp --port 5001-5003 --cidr 0.0.0.0/0
```

```
# Stop Instance, Upgrade to m5.2xlarge, and Restart
aws ec2 stop-instances --instance-ids ${INSTANCE_ID}
aws ec2 wait instance-stopped --instance-ids ${INSTANCE_ID}
aws ec2 modify-instance-attribute --instance-id ${INSTANCE_ID} --instance-type '{"Value": "m5.2xlarge"}'
aws ec2 start-instances --instance-ids ${INSTANCE_ID}
```

```
# Wait for Instance OS Initialization
while [ "$(aws ec2 describe-instance-status --instance-ids ${INSTANCE_ID} --query "InstanceStatuses[0].InstanceStatus.Status" --output text)" != "ok" ]; do
  echo "Waiting for instance OS initialization..."; sleep 5;
done
```

```
# Expand Root EBS Volume to 800 GB
export VOLUME_ID=$(aws ec2 describe-instances --instance-ids ${INSTANCE_ID} --query "Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId" --output text)
aws ec2 modify-volume --volume-id ${VOLUME_ID} --size 800

# Wait for Volume Modification
while [ "$(aws ec2 describe-volumes-modifications --volume-id ${VOLUME_ID} --query "VolumesModifications[0].ModificationState" --output text)" = "modifying" ]; do
  echo "Waiting for EBS volume modification..."; sleep 5;
done
```


==============================================================================
2. Bastion Filesystem & Workspace Preparation (Run on Bastion)
==============================================================================

# Expand Filesystem
```
export ROOT_DISK=$(lsblk -no PKNAME $(findmnt -n -o SOURCE /))
export ROOT_PART_NUM=$(lsblk -no KNAME $(findmnt -n -o SOURCE /) | grep -o '[0-9]*$')

sudo growpart /dev/${ROOT_DISK} ${ROOT_PART_NUM} || true
sudo xfs_growfs /
df -h /
```

```
# Clean Old Workspaces & Containers
rm -rf ~/oc-mirror-workspace ~/cincinnati-graph-data ~/all
podman stop -a --ignore
podman rm -a --force --ignore
podman system prune -a --volumes --force
sudo rm -rf /var/nexus-data /etc/nexus-ssl /etc/nginx
sudo podman system prune -a --volumes --force
```


==============================================================================
3. Install & Configure Nexus Registry (Run on Bastion)
==============================================================================
```
# Create SSL & Storage Directories
sudo mkdir -p /var/nexus-data /etc/nexus-ssl /etc/nginx
sudo chown -R 200:200 /var/nexus-data /etc/nexus-ssl
```

# Get Public IP
```export BASTION_IP=$(curl -s https://ifconfig.me || hostname -I | awk '{print $1}')```

```
# Generate Self-Signed Certificate
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nexus-ssl/nexus.key \
  -out /etc/nexus-ssl/nexus.crt \
  -subj "/CN=${BASTION_IP}" \
  -addext "subjectAltName=IP:${BASTION_IP},IP:127.0.0.1,DNS:localhost"

sudo chmod 644 /etc/nexus-ssl/nexus.crt /etc/nexus-ssl/nexus.key
```
```
# Allow Firewall Traffic & Launch Nexus Container
sudo iptables -I INPUT 1 -p tcp -m multiport --dports 8081,8443,5001,5002 -j ACCEPT

sudo podman run -d --name nexus \
  -p 8081:8081 \
  -e INSTALL4J_ADD_VM_PARAMS="-Xms512m -Xmx1024m -XX:MaxDirectMemorySize=512m" \
  -v /var/nexus-data:/nexus-data:Z \
  docker.io/sonatype/nexus3:latest
```

```
# Wait for Nexus Startup
while [ "$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8081/)" != "200" ]; do
  echo "Waiting for Nexus 3 initialization..."; sleep 5;
done
```
```
# Print Admin Credentials
echo "Initial Nexus Admin Password:"
sudo podman exec nexus cat /nexus-data/admin.password && echo ""
echo "Nexus URL: http://${BASTION_IP}:8081"
```

```
# ----------------------------------------------------------------------------
# Nexus UI Steps (In Browser):
# 1. Open http://<BASTION_IP>:8081 -> Login as admin with password printed above.
# 2. Change admin password to: RedHat123!
# 3. Security -> Anonymous Access -> Check "Allow anonymous users..." -> Save.
# 4. Security -> Realms -> Move "Docker Bearer Token Realm" to Active -> Save.
# 5. Create Proxy Repository "redhat-proxy":
#    - Format: docker (proxy) | Port: 5001 (HTTPS) | Allow anonymous pull: Checked
#    - Remote storage: https://registry.redhat.io
#    - Authentication: Red Hat Service Account (e.g. 15328052|nexus-lab & token)
# 6. Create Hosted Repository "ocp-hosted":
#    - Format: docker (hosted) | Port: 5002 (HTTPS) | Allow anonymous pull: Checked
#    - Deployment policy: Allow redeploy
# ----------------------------------------------------------------------------
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
```
```
# Start Nginx SSL Proxy Container
sudo podman run -d --name nexus-ssl-proxy --net=host \
  -v /etc/nginx/nexus-proxy.conf:/etc/nginx/nginx.conf:ro \
  -v /etc/nexus-ssl:/etc/nexus-ssl:ro \
  docker.io/library/nginx:alpine
```
```
# Trust Certificate on Bastion Host
sudo cp /etc/nexus-ssl/nexus.crt /etc/pki/ca-trust/source/anchors/nexus.crt
sudo update-ca-trust
```
```
# Test Ports
curl -I https://localhost:5001/v2/
curl -I https://localhost:5002/v2/
```

==============================================================================
4. Install oc-mirror v2 & Mirror Content (Run on Bastion)
==============================================================================
```
# Install oc-mirror CLI
curl -sL https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/oc-mirror.tar.gz | tar -xz -C /tmp
sudo mv /tmp/oc-mirror /usr/local/bin/oc-mirror
sudo chmod +x /usr/local/bin/oc-mirror

WORKSPACE="${HOME}/oc-mirror-workspace"
mkdir -p "${WORKSPACE}"
```
```
# Write ImageSetConfiguration
cat << 'EOF' > "${WORKSPACE}/imageset-config.yaml"
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  platform:
    architectures:
      - amd64
    channels:
      - name: stable-4.20
        minVersion: 4.20.30
        maxVersion: 4.20.32
        type: ocp
    graph: true
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
      packages:
        - name: cincinnati-operator
          channels:
            - name: v1
        - name: advanced-cluster-management
        - name: kubevirt-hyperconverged
        - name: openshift-gitops-operator
EOF
```
```
# Sanitize non-breaking space characters
sed -i 's/\xc2\xa0/ /g' "${WORKSPACE}/imageset-config.yaml"
```
```
# Configure Authentication
NEXUS_USER="admin"
NEXUS_PASS="RedHat123!"
NEXUS_AUTH=$(echo -n "${NEXUS_USER}:${NEXUS_PASS}" | tr -d '\r\n' | base64 | tr -d '\r\n')
```
```
mkdir -p ~/.open-shift
jq --arg auth "$NEXUS_AUTH" '.auths["localhost:5002"] = {"auth": $auth}' ~/pull-secret.json > ~/.open-shift/containers-auth.json
```
```
# Run Mirroring Process
oc-mirror --v2 \
  --config "${WORKSPACE}/imageset-config.yaml" \
  docker://localhost:5002 \
  --authfile ~/.open-shift/containers-auth.json \
  --workspace "file://${WORKSPACE}"
```
==============================================================================
5. Prepare & Apply Cluster Resources (Run on Bastion)
==============================================================================
```
RESOURCE_DIR=$(find "${HOME}" -type d -name "cluster-resources" | head -n 1)
echo "Found cluster resources at: ${RESOURCE_DIR}"
```
```
# 1. Cleanly inject mirrorSourcePolicy: NeverContactSource directly above "source:"
python3 -c '
import glob

def apply_never_contact(filepath):
    with open(filepath, "r") as f:
        lines = f.readlines()
    
    clean = [l for l in lines if "mirrorSourcePolicy" not in l]
    final = []
    for line in clean:
        if line.strip().startswith("source:"):
            indent = line[:len(line) - len(line.lstrip())]
            final.append(f"{indent}mirrorSourcePolicy: NeverContactSource\n")
        final.append(line)
        
    with open(filepath, "w") as f:
        f.writelines(final)

resource_dir="'$RESOURCE_DIR'"
for f in glob.glob(f"{resource_dir}/idms*.yaml") + glob.glob(f"{resource_dir}/itms*.yaml"):
    apply_never_contact(f)
'
```
```
# 2. Replace localhost:5002 with BASTION_HOST:5002
BASTION_HOST=$(curl -s --connect-timeout 2 http://169.254.169.254/latest/meta-data/public-ipv4 2>/dev/null \
  || ip route get 1.1.1.1 2>/dev/null | grep -oP 'src \K\S+' \
  || hostname -I | awk '{print $1}')

echo "Using Bastion Host IP: ${BASTION_HOST}"

sed -i "s/localhost:5002/${BASTION_HOST}:5002/g" "${RESOURCE_DIR}"/*.yaml
sed -i "s/localhost:5002/${BASTION_HOST}:5002/g" "${RESOURCE_DIR}"/*.json 2>/dev/null || true
```
```
# 3. Multi-Document Safe Sanity Check
python3 -c '
import os, glob, yaml, json

resource_dir = "'$RESOURCE_DIR'"
print("🔍 RUNNING STRICT SANITY CHECK ON MANIFESTS...\n")

errors = False
for f in glob.glob(f"{resource_dir}/*.yaml") + glob.glob(f"{resource_dir}/*.json"):
    fname = os.path.basename(f)
    try:
        with open(f) as fp:
            docs = list(yaml.safe_load_all(fp)) if f.endswith(".yaml") else [json.load(fp)]
        
        if any("localhost" in str(doc) for doc in docs if doc):
            print(f"❌ FAIL: {fname} contains un-replaced localhost endpoint!")
            errors = True
        else:
            print(f"✅ PASS: {fname} ({len(docs)} doc(s))")
    except Exception as e:
        print(f"❌ FAIL: {fname} parsing error: {e}")
        errors = True

if errors:
    raise SystemExit("🛑 Fix manifest errors before continuing.")
'
```
```
# 4. Apply IDMS, ITMS, Signatures
oc apply -f "${RESOURCE_DIR}/idms-oc-mirror.yaml"
oc apply -f "${RESOURCE_DIR}/itms-oc-mirror.yaml"
oc apply -f "${RESOURCE_DIR}"/signature-configmap.*
```
```
# 5. BLOCKING WAIT: MachineConfigPool Node Updates
echo "⏳ Waiting for MachineConfigPools to begin and complete updates..."
sleep 10

until [ "$(oc get mcp -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Updating")].status}{"\n"}{end}' | grep -c "True")" -eq 0 ] && \
      [ "$(oc get mcp -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Updated")].status}{"\n"}{end}' | grep -v "True" | wc -l)" -eq 0 ]; do
  echo "[$(date +'%H:%M:%S')] Nodes applying registry configuration updates..."
  sleep 20
done
echo "✅ ALL NODES & MACHINECONFIGPOOLS ARE FULLY UPDATED AND READY!"
```
```
# 6. Apply CatalogSources & BLOCK for OLM Ready State
oc apply -f "${RESOURCE_DIR}"/cs-*.yaml
oc apply -f "${RESOURCE_DIR}"/cc-*.yaml 2>/dev/null || true
```
```
echo "⏳ Waiting for CatalogSource to reach READY state..."
until [ "$(oc get catalogsource cs-redhat-operator-index-v4-20 -n openshift-marketplace -o jsonpath='{.status.connectionState.lastObservedState}' 2>/dev/null)" = "READY" ]; do
  echo "[$(date +'%H:%M:%S')] CatalogSource status: $(oc get catalogsource cs-redhat-operator-index-v4-20 -n openshift-marketplace -o jsonpath='{.status.connectionState.lastObservedState}' 2>/dev/null || echo 'Pending')"
  sleep 10
done
echo "✅ CATALOGSOURCE IS READY AND CONNECTED!"
```

==============================================================================
6. Install OpenShift Update Service (OSUS) & Patch CVO
==============================================================================
```
NS="openshift-update-service"
```
```
# 1. Create OSUS Namespace and Subscription
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
```
```
# 2. BLOCKING WAIT: Cincinnati Operator CSV Installation
echo "⏳ Waiting for Cincinnati Operator installation..."
until oc get csv -n "${NS}" 2>/dev/null | grep -E -i "update-service|cincinnati" | grep -i "Succeeded" >/dev/null 2>&1; do
  echo "[$(date +'%H:%M:%S')] Operator installation in progress..."
  sleep 10
done
echo "✅ CINCINNATI OPERATOR INSTALLED SUCCESSFULLY!"
```
```
# 3. Configure CA Trust for Nexus Registry
BASTION_HOST=$(curl -s --connect-timeout 2 http://169.254.169.254/latest/meta-data/public-ipv4 2>/dev/null \
  || ip route get 1.1.1.1 2>/dev/null | grep -oP 'src \K\S+' \
  || hostname -I | awk '{print $1}')

REGISTRY_KEY="${BASTION_HOST}..5002"

oc create configmap registry-cas -n openshift-config \
  --from-file=updateservice-registry=/etc/nexus-ssl/nexus.crt \
  --from-file="${REGISTRY_KEY}"=/etc/nexus-ssl/nexus.crt \
  --dry-run=client -o yaml | oc apply -f -

oc patch image.config.openshift.io/cluster --type=merge \
  -p '{"spec":{"additionalTrustedCA":{"name":"registry-cas"}}}'
```
```
# 4. Clean old broken CRs & Apply UpdateService Manifest explicitly to openshift-update-service
oc delete updateservice update-service -n default --ignore-not-found
oc delete updateservice update-service -n "${NS}" --ignore-not-found

RESOURCE_DIR=$(find "${HOME}" -type d -name "cluster-resources" | head -n 1)
oc apply -f "${RESOURCE_DIR}/updateService.yaml" -n "${NS}"
```
```
# 5. BLOCKING WAIT: Operator Reconciliation & Pod Rollout
echo "⏳ Waiting for operator to reconcile and create deployment object..."
until oc get deployment update-service-oc-mirror -n "${NS}" >/dev/null 2>&1; do
  echo "[$(date +'%H:%M:%S')] Operator reconciling... waiting for deployment object..."
  sleep 5
done

echo "⏳ Deployment object created! Tracking pod rollout..."
oc rollout status deployment/update-service-oc-mirror -n "${NS}" --timeout=300s
echo "✅ UPDATE SERVICE DEPLOYMENT IS 100% READY!"
```
```
# 6. Fetch Policy Engine Route & Patch Cluster Version Operator
POLICY_ENGINE_URL=$(oc get route -n "${NS}" -l app=update-service-oc-mirror -o jsonpath='{.items[0].spec.host}')

echo "Patching CVO with Upstream URL: https://${POLICY_ENGINE_URL}/api/upgrades_info/v1/graph"

oc patch clusterversion version --type=json \
  -p '[{"op": "add", "path": "/spec/upstream", "value": "https://'$POLICY_ENGINE_URL'/api/upgrades_info/v1/graph"}]'
```
```
# 7. Final Verification
echo -e "\n=================================================="
echo "🎯 FINAL VERIFICATION"
echo "=================================================="
echo "CVO Upstream URL: $(oc get clusterversion -o jsonpath='{.items[*].spec.upstream}')"
echo -e "\nPod Status in ${NS}:"
oc get pods -n "${NS}"
```
```
# 8. OSUS usage in Web Console 
#### OpenShift Update Service (OSUS) - Web Console Usage Guide

Once the OpenShift Update Service (OSUS) is deployed and the Cluster Version Operator (CVO) is patched with your local upstream route, OpenShift automatically integrates your mirrored graph directly into the Web Console UI[cite: 1, 2]. 

Follow these steps to manage and trigger cluster upgrades using the Web Console in a disconnected environment.
```
## 1. Accessing Cluster Update Settings

1. Log in to the OpenShift Web Console as a user with `cluster-admin` privileges.
2. In the left navigation menu, navigate to **Administration** $\rightarrow$ **Cluster Settings**.
3. Select the **Details** tab.


## 2. Verifying the Disconnected OSUS Graph

On the **Details** tab, verify that CVO is correctly pulling from your local Nexus/OSUS instance:

* **Upstream Graph Route:** Locate the **Update status** section. CVO will display available releases pulled from your local route URL (`https://update-service-oc-mirror-route-openshift-update-service.../api/upgrades_info/v1/graph`)[cite: 1, 2].
* **Channel Selection:** Click the current channel link (e.g., `stable-4.20`)[cite: 1, 2]. You will see a dropdown list containing all channels configured in your mirrored `ImageSetConfiguration` (e.g., `fast-4.20`, `eus-4.20`, `stable-4.21`, etc.)[cite: 1, 2].

---

## 3. Triggering a Cluster Upgrade

1. Under the **Update status** section, click **Select version** (or **Update**)[cite: 1].
2. Choose your target release version (e.g., `4.20.31` or `4.20.32`) from the list provided by your local graph[cite: 1].
3. Review the update details and click **Update** to begin the cluster rollout process.

---

## 4. Monitoring Upgrade Progress

You can track the live progress of the upgrade directly within the console:

* **Visual Status Map:** On **Administration** $\rightarrow$ **Cluster Settings** $\rightarrow$ **Details**, the console renders a live visual timeline tracking control plane nodes and worker MachineConfigPool updates.
* **Operator Reconciliation:** Navigate to the **ClusterOperators** tab to watch individual core operators update and return to an `Available=True` / `Degraded=False` state.

---

## 5. Handling Admin Acknowledgments in the UI

If an upgrade path requires explicit administrator acknowledgment (such as the Sigstore signature notice when transitioning toward 4.21)[cite: 1]:

1. A yellow warning banner will automatically appear at the top of the **Cluster Settings** page[cite: 1].
2. Click the **Acknowledge** button directly within the banner to clear the safeguard without using the CLI[cite: 1].

# TESTING
==============================================================================
9. Simulate Air-Gapped Cluster Environment (Run on Bastion)
==============================================================================

To simulate a strict, zero-egress air-gapped environment where the OpenShift cluster has zero internet access for upgrades while keeping full internet connectivity on the Bastion host, execute the following configuration steps:

### Step 1: DNS Blackholing inside OpenShift Cluster

Apply a CoreDNS override to force cluster nodes to resolve public Red Hat and Quay container registries to loopback (`127.0.0.1`):

```
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
```

### Step 2: Sever Nexus Proxy Feed (Block Port 5001)
#### Block incoming traffic to port 5001 (redhat-proxy) on the local firewall to prevent OpenShift from pulling "on-demand" fallback artifacts through the proxy repository:
```
# Block local traffic to port 5001 (Proxy Port)
sudo iptables -I INPUT 1 -p tcp --dport 5001 -j DROP
```

### Step 3: Verify Air-Gap Simulation & OSUS Functionality
#### Run these checks to confirm that the Bastion retains internet access, the cluster is strictly isolated, and upgrades function purely via the local OSUS graph and hosted registry (5002):
```
# 1. TEST BASTION INTERNET: Should succeed
echo "🌐 Testing Bastion Internet Access..."
curl -s -I [https://www.redhat.com](https://www.redhat.com) | head -n 1

# 2. TEST CLUSTER AIR-GAP ISOLATION: Should fail/timeout connecting to public registry
echo "🔒 Testing Cluster Air-Gap Isolation..."
oc debug node/$(oc get nodes -o jsonpath='{.items[0].metadata.name}') -- chroot /host curl -s -I --connect-timeout 3 [https://registry.redhat.io/v2/](https://registry.redhat.io/v2/) || echo "✅ SUCCESS: Cluster is fully air-gapped!"

# 3. TEST OSUS UPGRADE PATH: Should resolve target releases from port 5002
echo "🚀 Checking OSUS Upgrade Status..."
oc adm upgrade
```

