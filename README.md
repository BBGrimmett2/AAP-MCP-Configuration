# AAP MCP Server Tool Filtering - ArgoCD Example

This directory contains an ArgoCD-compatible example for deploying custom MCP tool filtering configurations.

## How It Works

1. **ConfigMap** - Contains custom `aap-mcp.yaml` with filtered toolsets
2. **Deployment Patch** - Strategic merge patch adds volume mount
3. **Kustomize** - Combines base resources with overlay patches
4. **ArgoCD** - Continuously reconciles the desired state

The deployment patch adds:
- A volume referencing the ConfigMap
- A volumeMount overlaying `/app/aap-mcp.yaml` in the container

## Why This Survives Reconciliation

The AAP MCP Server operator uses **server-side apply** which:
- Merges changes from multiple sources
- Preserves fields not explicitly managed by the operator
- Only reconciles fields defined in its template

Our custom volume and mount are **additive** (not replacing operator-managed fields), so they persist across reconciliation cycles.

**Tested:** Changing `spec.no_log` or other CR fields triggers operator reconciliation, and the custom volume mount remains intact.

## Directory Structure

```
.
├── README.md                       
├── base/
│   ├── kustomization.yaml          # Base kustomization
│   ├── configmap.yaml              # Custom aap-mcp.yaml config
│   └── patch-deployment.yaml       # Strategic merge patch for volume mount
└── overlays/
    ├── minimal/                    # Minimal toolset (1 tool)
    │   ├── kustomization.yaml
    │   └── configmap.yaml
    └── restricted-example/         # Custom toolset configuration
        ├── kustomization.yaml
        └── configmap.yaml
```

## Testing Configuration

Test this configuration with the following commands:

```bash
# Apply the minimal overlay (1 tool only)
kubectl apply -k overlays/minimal/

# Verify ConfigMap was created
kubectl get configmap aap-mcp-custom-config -n aap-operator

# Verify deployment was patched
kubectl get deployment aap-mcp-server -n aap-operator \
  -o jsonpath='{.spec.template.spec.volumes[?(@.name=="custom-mcp-config")]}' | jq .

# Wait for pod to restart
kubectl rollout status deployment/aap-mcp-server -n aap-operator

# Check configuration loaded in pod
kubectl exec -n aap-operator deployment/aap-mcp-server -- \
  cat /app/aap-mcp.yaml | grep -A 20 "toolsets:"

# Check MCP server logs for tool count (should show only 1 tool)
kubectl logs -n aap-operator deployment/aap-mcp-server --tail=50 | \
  grep -E "Toolsets:|job_management:|Total tools"

# Expected output:
#   Toolsets: 3 enabled
#   Total tools loaded: 1041
#   job_management: 1
#   all: 1

# Test reconciliation survival
echo "Testing operator reconciliation..."
kubectl patch ansiblemcpserver aap-mcp-server -n aap-operator \
  --type=merge -p '{"spec":{"no_log":false}}'

sleep 15

# Verify custom config still present after reconciliation
kubectl get deployment aap-mcp-server -n aap-operator \
  -o jsonpath='{.spec.template.spec.volumes[?(@.name=="custom-mcp-config")]}' | jq .

# Revert no_log change
kubectl patch ansiblemcpserver aap-mcp-server -n aap-operator \
  --type=merge -p '{"spec":{"no_log":true}}'

# Test restricted (read-only) overlay
kubectl apply -k overlays/restricted-example/

kubectl rollout status deployment/aap-mcp-server -n aap-operator

kubectl logs -n aap-operator deployment/aap-mcp-server --tail=50 | \
  grep -E "Toolsets:|Total tools"

# Clean up (revert to default)
kubectl delete configmap aap-mcp-custom-config -n aap-operator
kubectl rollout restart deployment/aap-mcp-server -n aap-operator
```

## Overlays

### `overlays/minimal`

Minimal configuration with only 1 tool enabled:
- `controller.job_templates_launch_retrieve`

Use this as a starting point to selectively enable tools.

### `overlays/restricted-example`

Read-only configuration with all write operations disabled:
- All `_create`, `_update`, `_destroy`, `_delete`, `_partial_update` tools removed
- Only query and retrieve operations available
- Suitable for read-only AI assistants or monitoring tools

## Customizing Tool Selection

1. Copy an overlay or create a new one:

```bash
cp -r overlays/minimal overlays/my-config
```

2. Edit `overlays/my-config/configmap.yaml` to enable/disable tools:

```yaml
toolsets:
  job_management:
    - controller.job_templates_list
    - controller.job_templates_retrieve
    # - controller.job_templates_create  # Disabled
```

3. Update `kustomization.yaml` to reference your config:

```yaml
bases:
  - ../../base
patchesStrategicMerge:
  - configmap.yaml
```

4. Apply your overlay:

```bash
kubectl apply -k overlays/my-config/
```

## Verification

After applying, verify the configuration:

```bash
# Check ConfigMap exists
kubectl get configmap aap-mcp-custom-config -n aap-operator

# Check volume mount in deployment
kubectl get deployment aap-mcp-server -n aap-operator \
  -o jsonpath='{.spec.template.spec.volumes[?(@.name=="custom-mcp-config")]}'

# Check config loaded in pod
kubectl exec -n aap-operator deployment/aap-mcp-server -- \
  cat /app/aap-mcp.yaml | head -60

# Check MCP server logs for tool count
kubectl logs -n aap-operator deployment/aap-mcp-server --tail=50 | grep -E "Toolsets:|Total tools"
```

Expected output:
```
Toolsets: X enabled
Total tools loaded: 1041
  job_management: Y
  inventory_management: Z
  ...
```

## Monitoring Reconciliation

Watch for operator reconciliation events:

```bash
# Watch deployment changes
kubectl get deployment aap-mcp-server -n aap-operator -w

# Check operator logs
kubectl logs -n aap-operator -l app.kubernetes.io/name=ansible-mcp-server-operator --tail=100 -f
```

If the custom volume mount disappears, it means:
- Operator changed its reconciliation strategy
- The patch is incompatible with a new operator version
- You should open an issue with Red Hat

## Troubleshooting

### ConfigMap Not Applied

**Symptom:** Deployment has no custom-mcp-config volume

**Resolution:**
```bash
# Check if patch was applied
kubectl get deployment aap-mcp-server -n aap-operator -o yaml | grep custom-mcp-config

# Manually apply the patch
kubectl patch deployment aap-mcp-server -n aap-operator --type=strategic --patch-file base/patch-deployment.yaml
```

### Tools Still All Available

**Symptom:** All 1041 tools available despite custom config

**Causes:**
1. ConfigMap not mounted
2. Pod not restarted
3. YAML syntax error

**Resolution:**
```bash
# Verify config loaded
kubectl exec -n aap-operator deployment/aap-mcp-server -- cat /app/aap-mcp.yaml

# Restart pod
kubectl rollout restart deployment/aap-mcp-server -n aap-operator
```

