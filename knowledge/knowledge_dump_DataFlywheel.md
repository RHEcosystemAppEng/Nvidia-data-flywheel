# Data Flywheel Blueprint - Troubleshooting Guide

This document provides troubleshooting guidance for NVIDIA Data Flywheel Blueprint v0.3.0 deployment on OpenShift with existing NeMo Microservices infrastructure. These issues were encountered during real-world deployments and the solutions have been integrated into the automated deployment script.

## When to Use This Guide

Use this guide if:
- You're deploying Data Flywheel on OpenShift with pre-deployed NeMo Microservices v25.08
- You're experiencing evaluation failures (base-eval, icl-eval) with pre-deployed NIMs
- Customization (fine-tuning) jobs are failing with permission denied errors
- You need to redeploy or upgrade NeMo Microservices and maintain Data Flywheel functionality
- You need to understand what the automated deployment script does behind the scenes

## Common Scenario

Deploying Data Flywheel Blueprint to work with existing NeMo Microservices infrastructure where NIMs are deployed via NeMo Operator rather than through NeMo Microservices deployment APIs.

**Example Environment:**
- **Platform**: OpenShift cluster with GPU nodes
- **NeMo Microservices**: v25.08 (pre-deployed via upstream Helm chart)
- **NIMs**: Deployed via NeMo Operator (e.g., `meta/llama-3.2-1b-instruct`)
- **Data Flywheel**: v0.3.0
- **Infrastructure**: Elasticsearch, MongoDB, Redis, NeMo Gateway (deployed by Data Flywheel prerequisites)

---

## Error Categories and Fixes

### 1. Base Model Evaluations Failing (404 Not Found)

**Error:**
```
404, message='Not Found', url='http://nemoentitystore-sample.hacohen-flywheel.svc.cluster.local:8000/v1/models/meta/llama-3.2-1b-instruct'
```

or

```
ValidationError: namespace="" fails pattern ^[\w\-.]+$
```

or

```
404 Not Found: The model `default/meta-llama-3.2-1b-instruct` does not exist
```

**Root Cause:**
- NeMo Evaluator queries Entity Store for model metadata before running evaluations
- For customized models, metadata exists in Entity Store (created during customization)
- For base NIMs deployed via NeMo Operator, metadata doesn't exist in Entity Store
- Evaluator defaults to querying Entity Store directly at `http://nemoentitystore-sample:8000`
- Entity Store returns 404 because the base model was never registered

**Why This Happens After NeMo Microservices Redeployment:**
- The upstream NeMo Microservices Helm chart sets `ENTITY_STORE_URL` to the default Entity Store service
- When you redeploy NeMo Microservices, this environment variable is reset to default
- Previous configuration pointing to the gateway is lost

**Fix Applied:**
Created NeMo Gateway with mock Entity Store endpoints in [deploy/flywheel-prerequisites/templates/nemo-gateway-configmap.yaml](../deploy/flywheel-prerequisites/templates/nemo-gateway-configmap.yaml):

```nginx
# Mock Entity Store endpoint for base NIMs
location ~ ^/v1/models/meta/llama-3\.2-1b-instruct$ {
    default_type application/json;
    return 200 '{
        "name": "llama-3.2-1b-instruct",
        "namespace": "meta",
        "base_model": "meta/llama-3.2-1b-instruct",
        "display_name": "Llama 3.2 1B Instruct",
        "model_type": "llm",
        "task_type": "chat",
        "schema_version": "1.0.0"
    }';
}
```

**Configuration Applied:**
The `make install-flywheel` target in [deploy/Makefile](../deploy/Makefile) automatically configures NeMo Evaluator:

```bash
# Configure NeMo Evaluator to use gateway for Entity Store
echo "🔧 Configuring NeMo Evaluator to use gateway..."
if oc get deployment nemoevaluator-sample -n "$NAMESPACE" &>/dev/null; then
    oc set env deployment/nemoevaluator-sample -n "$NAMESPACE" ENTITY_STORE_URL="http://nemo-gateway"
    echo "   ✓ NeMo Evaluator configured to use gateway"

    echo "   Waiting for NeMo Evaluator to restart..."
    oc rollout status deployment/nemoevaluator-sample -n "$NAMESPACE" --timeout=120s || true
fi
```

**Manual Fix:**
If you redeploy NeMo Microservices, reconfigure the evaluator:

```bash
# Configure NeMo Evaluator to use gateway
oc set env deployment/nemoevaluator-sample -n $NAMESPACE ENTITY_STORE_URL="http://nemo-gateway"

# Wait for NeMo Evaluator to restart
oc rollout status deployment/nemoevaluator-sample -n $NAMESPACE
```

**Verification:**
```bash
# Test the mock endpoint
oc run curl-test --rm -i --restart=Never --image=curlimages/curl:latest -n $NAMESPACE -- \
  curl -s http://nemo-gateway/v1/models/meta/llama-3.2-1b-instruct | jq .

# Check NeMo Evaluator is configured to use gateway
oc get deployment nemoevaluator-sample -n $NAMESPACE \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="ENTITY_STORE_URL")].value}'
# Should output: http://nemo-gateway
```

---

### 2. Evaluation Jobs Failing with Secret Deletion Error

**Error:**
```
500: Failed to delete secret. // ... User "system:serviceaccount:hacohen-flywheel:nemoevaluator-sample" cannot delete resource "secrets" in API group "" in the namespace "hacohen-flywheel"
```

**Root Cause:**
- NeMo Evaluator creates temporary Kubernetes secrets for evaluation job configurations
- After evaluation completes, the evaluator tries to clean up these secrets
- The upstream NeMo Microservices deployment creates a role for nemoevaluator with only: `get, watch, list, create` verbs
- The `delete` verb is missing, so cleanup fails with 403 Forbidden
- This causes the evaluation job to fail even though the actual evaluation completed successfully

**Why This Happens:**
- Upstream NeMo Microservices Helm chart defines the nemoevaluator role without delete permission
- This is likely an oversight in the upstream chart
- Every evaluation job creates secrets but can't clean them up

**Fix Applied:**
The `make install-flywheel` target in [deploy/Makefile](../deploy/Makefile) automatically fixes RBAC permissions:

```bash
# Fix RBAC permissions - upstream NeMo Microservices is missing delete permission for secrets
echo "   Fixing NeMo Evaluator RBAC permissions for secret cleanup..."
if oc get role nemoevaluator-sample -n "$NAMESPACE" &>/dev/null; then
    # Check if delete permission is missing
    if ! oc get role nemoevaluator-sample -n "$NAMESPACE" -o jsonpath='{.rules[1].verbs}' | grep -q "delete"; then
        oc patch role nemoevaluator-sample -n "$NAMESPACE" --type=json \
          -p='[{"op": "add", "path": "/rules/1/verbs/-", "value": "delete"}]'
        echo "   ✓ Added delete permission for secrets"
    else
        echo "   ✓ Delete permission already exists"
    fi
fi
```

**Manual Fix:**
```bash
# Add delete permission to the nemoevaluator role
oc patch role nemoevaluator-sample -n $NAMESPACE --type=json \
  -p='[{"op": "add", "path": "/rules/1/verbs/-", "value": "delete"}]'
```

**Verification:**
```bash
# Check the role has delete permission
oc get role nemoevaluator-sample -n $NAMESPACE -o jsonpath='{.rules[1].verbs}' | grep delete
```

---

### 3. Customization Jobs Failing with Permission Denied

**Error:**
```
Cannot read directory '/mount/models/llama32_1b-instruct_2_0'. Reason: [Errno 13] Permission denied
```

**Root Cause:**
- Base model files are stored in the `finetuning-ms-models-pvc` PersistentVolumeClaim
- When NeMo Microservices is first deployed, training pods run as UID 1000
- Model files are downloaded and owned by UID 1000:1000
- After redeploying NeMo Microservices, OpenShift assigns random UIDs from the namespace's UID range (e.g., 1000920000+)
- Even though the nemo-customizer-scc has `RunAsUser: RunAsAny`, pods don't explicitly set `runAsUser`
- OpenShift defaults to assigning a UID from the namespace range
- Training pods running as UID 1000920000 cannot read files owned by UID 1000

**Why This Happens After Redeployment:**
- Fresh NeMo Microservices deployment → new pod security contexts
- OpenShift assigns UIDs from namespace annotation: `openshift.io/sa.scc.uid-range`
- Previous file ownership (UID 1000) doesn't match new pod UIDs
- File permissions prevent training pods from reading base model weights

**Investigation Process:**
1. Checked SCC permissions - `nemo-customizer-scc` exists with `RunAsAny`
2. Verified model files exist in PVC - files present but owned by UID 1000
3. Checked training pod UID - running as 1000920000 (from namespace range)
4. Identified mismatch - pod UID ≠ file owner UID → permission denied

**Fix Applied:**
The `make install-flywheel` target in [deploy/Makefile](../deploy/Makefile) automatically fixes model file permissions:

```bash
# Fix base model file permissions for customization jobs
echo "🔧 Fixing base model file permissions for customization..."
if oc get pvc finetuning-ms-models-pvc -n "$NAMESPACE" &>/dev/null; then
    # Get the UID that OpenShift assigns to pods (from the namespace's UID range)
    OPENSHIFT_UID=$(oc get namespace "$NAMESPACE" \
      -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}' | cut -d'/' -f1)

    if [ -n "$OPENSHIFT_UID" ]; then
        echo "   Detected OpenShift UID range starting at: $OPENSHIFT_UID"
        echo "   Updating model file ownership to match OpenShift UID..."

        # Run a temporary pod to fix permissions
        oc run fix-model-perms --rm -i --restart=Never --image=busybox:latest -n "$NAMESPACE" \
            --overrides="{\"spec\":{\"securityContext\":{\"runAsUser\":0},\"containers\":[{\"name\":\"fix\",\"image\":\"busybox\",\"command\":[\"sh\",\"-c\",\"chown -R $OPENSHIFT_UID:1000 /mount/models/* 2>/dev/null || true\"],\"volumeMounts\":[{\"name\":\"models\",\"mountPath\":\"/mount/models\"}]}],\"volumes\":[{\"name\":\"models\",\"persistentVolumeClaim\":{\"claimName\":\"finetuning-ms-models-pvc\"}}],\"serviceAccountName\":\"default\"}}" \
            --timeout=60s 2>/dev/null || true

        echo "   ✓ Model file permissions updated"
    fi
fi
```

**Manual Fix:**
```bash
# Get the OpenShift UID range for your namespace
OPENSHIFT_UID=$(oc get namespace $NAMESPACE \
  -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}' | cut -d'/' -f1)

# Fix model file ownership
oc run fix-model-perms --rm -i --restart=Never --image=busybox:latest -n $NAMESPACE \
  --overrides="{\"spec\":{\"securityContext\":{\"runAsUser\":0},\"containers\":[{\"name\":\"fix\",\"image\":\"busybox\",\"command\":[\"chown\",\"-R\",\"$OPENSHIFT_UID:1000\",\"/mount/models\"],\"volumeMounts\":[{\"name\":\"models\",\"mountPath\":\"/mount/models\"}]}],\"volumes\":[{\"name\":\"models\",\"persistentVolumeClaim\":{\"claimName\":\"finetuning-ms-models-pvc\"}}],\"serviceAccountName\":\"default\"}}"
```

**Verification:**
```bash
# Check customization job logs
oc logs -n $NAMESPACE -l job-name --tail=50 | grep -i "permission\|error"

# Or check a specific job
oc logs -n $NAMESPACE job/<customization-job-name> --tail=100
```

---

## Architecture: NeMo Gateway

The NeMo Gateway is a critical component that unifies access to NeMo Microservices and provides mock endpoints for base NIMs.

**Gateway Routes** ([deploy/flywheel-prerequisites/templates/nemo-gateway-configmap.yaml](../deploy/flywheel-prerequisites/templates/nemo-gateway-configmap.yaml)):

| Path | Target Service | Purpose |
|------|-------------------|---------|
| `/v1/datasets` | nemodatastore-sample | Dataset management |
| `/v1/customization` | nemocustomizer-sample | Fine-tuning jobs |
| `/v1/evaluation` | nemoevaluator-sample | Model evaluation |
| `/v1/guardrails` | nemoguardrails-sample | Safety filters |
| `/v1/entity-store` | nemoentitystore-sample | Model metadata (customized models) |
| `/v1/models/{namespace}/{name}` | Mock endpoint | Model metadata (base NIMs) |
| `/v1/namespaces` | nemoentitystore-sample | Namespace management |
| `/v1/datastore` | nemodatastore-sample | Datastore operations |
| `/v1/hf` | nemodatastore-sample | HuggingFace API |
| `/{namespace}/{repo}.git/` | nemodatastore-sample | Git/LFS operations |

**Mock Entity Store Endpoints:**
- Provide valid model metadata for base NIMs deployed via NeMo Operator
- Return Entity Store-compatible schema with correct `namespace` and `name` fields
- Enable base-eval and icl-eval to work without requiring model registration
- Each base NIM requires a separate mock endpoint definition

**Example Mock Endpoint:**
```nginx
location ~ ^/v1/models/meta/llama-3\.2-1b-instruct$ {
    default_type application/json;
    return 200 '{
        "name": "llama-3.2-1b-instruct",
        "namespace": "meta",
        "base_model": "meta/llama-3.2-1b-instruct",
        "display_name": "Llama 3.2 1B Instruct",
        "model_type": "llm",
        "task_type": "chat",
        "schema_version": "1.0.0"
    }';
}
```

**Why Mock Endpoints Are Needed:**
- NeMo Evaluator constructs model paths as `{namespace}/{name}` (e.g., `meta/llama-3.2-1b-instruct`)
- For customized models: Entity Store has real metadata from the customization process
- For base NIMs: No metadata exists because they were deployed directly via NeMo Operator
- Gateway intercepts requests for base NIMs and returns valid mock metadata
- This allows evaluator to proceed with base-eval and icl-eval

---

## Deployment Automation Structure

The `make install-flywheel` target in [deploy/Makefile](../deploy/Makefile) includes these phases:

### Phase 1: Deploy Infrastructure Prerequisites
1. Deploy flywheel-prerequisites Helm chart:
   - Elasticsearch (with anyuid SCC)
   - MongoDB
   - Redis
   - NeMo Gateway (nginx with mock Entity Store endpoints)
2. Wait for all infrastructure services to be ready
3. Validate each service (Elasticsearch, MongoDB, Redis, Gateway)

### Phase 2: Configure NeMo Microservices
1. **Configure NeMo Evaluator**:
   - Set `ENTITY_STORE_URL="http://nemo-gateway"`
   - Wait for evaluator deployment to restart
2. **Fix RBAC Permissions**:
   - Check if nemoevaluator role has delete permission
   - Add delete verb if missing
3. **Fix Model File Permissions**:
   - Detect OpenShift namespace UID range
   - Update model file ownership to match UID range
   - Allows training pods to read base model files

### Phase 3: Deploy Data Flywheel Services
1. Retrieve NGC API key from cluster secret
2. Add NeMo Microservices Helm repository
3. Update Data Flywheel chart dependencies
4. Install or upgrade Data Flywheel Helm chart with custom values
5. Configure OpenShift security (service accounts, SCC)
6. Update secrets (NVIDIA_API_KEY, HF_TOKEN)
7. Restart deployments to pick up configuration

### Phase 4: Verification
1. Check Data Flywheel pod status
2. List Data Flywheel services
3. Provide next steps for port-forwarding and running tutorial

---

## Key Configuration Files

### [deploy/flywheel-components/values-openshift.yaml](../deploy/flywheel-components/values-openshift.yaml)

Configures Data Flywheel for OpenShift with existing infrastructure:

```yaml
# Disable embedded infrastructure (use flywheel-prerequisites instead)
elasticsearch:
  enabled: false

mongodb:
  enabled: false

redis:
  enabled: false

nemo-microservices:
  enabled: false  # Use existing NeMo Microservices

# Point to existing infrastructure services
env:
  ELASTICSEARCH_URL: "https://elasticsearch-master:9200"
  ELASTICSEARCH_USER: "elastic"
  MONGODB_URL: "mongodb://flywheel-infra-mongodb:27017"
  REDIS_URL: "redis://flywheel-infra-redis-master:6379/0"
  HOME: "/tmp"  # Writable directory for uv cache (OpenShift requirement)

# NeMo Microservices configuration
nmp_config:
  nemo_base_url: "http://nemo-gateway"  # Unified gateway
  nim_base_url: "http://meta-llama3-1b-instruct:8000"  # NIM deployed via NeMo Operator
  datastore_base_url: "http://nemo-gateway"
  nmp_namespace: "hacohen-flywheel"
```

### [deploy/flywheel-prerequisites/values.yaml](../deploy/flywheel-prerequisites/values.yaml)

Configures infrastructure services:

```yaml
namespace:
  create: false  # Namespace already exists

elasticsearch:
  enabled: true
  replicas: 1
  minimumMasterNodes: 1

mongodb:
  enabled: true
  architecture: standalone

redis:
  enabled: true
  architecture: standalone

nemo-gateway:
  enabled: true
  image:
    repository: nginx
    tag: "1.25-alpine"
```

---

## Common Patterns and Learnings

### 1. NeMo Microservices Redeployment Impact

**Problem:** Redeploying NeMo Microservices from upstream resets critical configurations:
- `ENTITY_STORE_URL` reverts to default Entity Store URL
- RBAC permissions may be reset (missing delete verb)
- Pod UIDs change, breaking file access

**Solution:** Always run the Data Flywheel deployment after NeMo Microservices redeployment:
```bash
cd /path/to/Nvidia-data-flywheel/deploy
make install-flywheel
```

The Makefile detects existing installations and reconfigures without reinstalling.

### 2. OpenShift UID Range Management

**Problem:** OpenShift assigns UIDs from namespace-specific ranges, not fixed UIDs.

**Detection:**
```bash
oc get namespace $NAMESPACE \
  -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}'
# Example output: 1000920000/10000
```

**Solution:** File ownership must match the namespace UID range, not fixed UID 1000.

### 3. Entity Store vs Mock Endpoints

**Customized Models:**
- Created through NeMo Customizer API
- Metadata automatically registered in Entity Store
- Evaluator queries Entity Store directly
- Model path: `{namespace}/customized-{base-model}@{job-id}`

**Base NIMs (NeMo Operator):**
- Deployed as standalone Kubernetes services
- No metadata in Entity Store
- Evaluator must use gateway mock endpoints
- Model path: `{namespace}/{name}` (e.g., `meta/llama-3.2-1b-instruct`)

### 4. Service Account Permissions

**NeMo Evaluator Requirements:**
- Create secrets (evaluation job configuration)
- Delete secrets (cleanup after evaluation)
- Get/watch/list secrets (monitoring)

**Upstream NeMo Microservices Role:**
- Only grants: get, watch, list, create
- Missing: delete

**Fix:** Add delete verb to role after deployment.

### 5. Data Flywheel Workflow Verification

**Complete workflow test sequence:**
1. **base-eval**: Evaluate base NIM performance
2. **icl-eval**: Evaluate with in-context learning examples
3. **customization**: Fine-tune the model
4. **customized-eval**: Evaluate fine-tuned model performance

**Success indicators:**
- All evaluation types complete without errors
- Customization job runs to completion (multiple epochs)
- Customized model shows improved metrics vs base model
- Customized model registered in Entity Store

**Example successful metrics progression:**
```
base-eval:       function_name=0.20, tool_calling_correctness=0.07
icl-eval:        function_name=0.30, tool_calling_correctness=0.22
customized-eval: function_name=0.45, tool_calling_correctness=0.27
```

---

## Troubleshooting Checklist

### Before Redeploying NeMo Microservices

1. **Document current state**:
   ```bash
   # Save evaluator configuration
   oc get deployment nemoevaluator-sample -n $NAMESPACE -o yaml > nemoevaluator-backup.yaml

   # Check current UID range
   oc get namespace $NAMESPACE -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}'
   ```

2. **Verify Data Flywheel is working**:
   ```bash
   # Check all pods running
   oc get pods -n $NAMESPACE | grep "^df-"

   # Test API
   oc port-forward -n $NAMESPACE svc/df-api-service 8000:8000 &
   curl http://localhost:8000/health
   ```

### After Redeploying NeMo Microservices

1. **Run Data Flywheel deployment**:
   ```bash
   cd /path/to/Nvidia-data-flywheel/deploy
   make install-flywheel
   ```

2. **Verify configurations**:
   ```bash
   # Check ENTITY_STORE_URL
   oc get deployment nemoevaluator-sample -n $NAMESPACE \
     -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="ENTITY_STORE_URL")].value}'

   # Check RBAC permissions
   oc get role nemoevaluator-sample -n $NAMESPACE \
     -o jsonpath='{.rules[1].verbs}' | grep delete

   # Check model file ownership
   OPENSHIFT_UID=$(oc get namespace $NAMESPACE \
     -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}' | cut -d'/' -f1)
   echo "Expected UID: $OPENSHIFT_UID"
   ```

3. **Test end-to-end workflow**:
   ```bash
   # Run notebook or API test
   # Should complete: base-eval → icl-eval → customization → customized-eval
   ```

### If Evaluations Still Fail

1. **Check gateway connectivity**:
   ```bash
   oc run curl-test --rm -i --restart=Never --image=curlimages/curl:latest -n $NAMESPACE -- \
     curl -s http://nemo-gateway/v1/models/meta/llama-3.2-1b-instruct
   ```

2. **Check evaluator logs**:
   ```bash
   oc logs -n $NAMESPACE deployment/nemoevaluator-sample --tail=100
   ```

3. **Check Celery worker logs**:
   ```bash
   oc logs -n $NAMESPACE deployment/df-celery-worker-deployment --tail=100
   ```

4. **Monitor Flower UI**:
   ```bash
   oc port-forward -n $NAMESPACE svc/df-flower-service 5555:5555 &
   # Open http://localhost:5555 in browser
   ```

### If Customization Jobs Fail

1. **Check training pod logs**:
   ```bash
   oc logs -n $NAMESPACE -l job-name --tail=100
   ```

2. **Check file permissions in PVC**:
   ```bash
   oc run debug-perms --rm -i --restart=Never --image=busybox:latest -n $NAMESPACE \
     --overrides='{"spec":{"containers":[{"name":"debug","image":"busybox","command":["ls","-la","/mount/models"],"volumeMounts":[{"name":"models","mountPath":"/mount/models"}]}],"volumes":[{"name":"models","persistentVolumeClaim":{"claimName":"finetuning-ms-models-pvc"}}]}}'
   ```

3. **Check pod UID**:
   ```bash
   oc get pod -n $NAMESPACE -l job-name -o jsonpath='{.items[0].status.containerStatuses[0].state.running.startedAt}'
   oc debug node/<node-name> -- chroot /host ps aux | grep customization
   ```

---

## References

- [deploy/Makefile](../deploy/Makefile) - Main deployment automation
- [deploy/flywheel-components/values-openshift.yaml](../deploy/flywheel-components/values-openshift.yaml) - Data Flywheel OpenShift configuration
- [deploy/flywheel-prerequisites/](../deploy/flywheel-prerequisites/) - Infrastructure Helm chart
- [deploy/flywheel-prerequisites/templates/nemo-gateway-configmap.yaml](../deploy/flywheel-prerequisites/templates/nemo-gateway-configmap.yaml) - Gateway configuration with mock endpoints
- [knowledge_dump_demo_workflow.md](knowledge_dump_demo_workflow.md) - Demo workflow guide and validation
- [.env.example](../.env.example) - Environment variables template

---

## Summary

The Data Flywheel Blueprint deployment on OpenShift with existing NeMo Microservices requires three critical post-deployment configurations:

1. **ENTITY_STORE_URL Configuration**: Point evaluator to gateway for base NIM metadata
2. **RBAC Permission Fix**: Add delete verb to nemoevaluator role for secret cleanup
3. **Model File Permissions**: Update ownership to match OpenShift UID range for training pods

These configurations are automatically applied by the deployment script and must be reapplied whenever NeMo Microservices is redeployed from upstream.

The NeMo Gateway serves as a unified proxy to all NeMo services and provides mock Entity Store endpoints for base NIMs, eliminating the need for manual model registration and enabling seamless evaluation workflows.
