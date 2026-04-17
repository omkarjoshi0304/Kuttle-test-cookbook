

  

A beginner-friendly guide to writing Kubernetes operator tests using KUTTL (KUbernetes Test TooL).

  

## Table of Contents

- [What is KUTTL?](#what-is-kuttl)

- [Basic Concepts](#basic-concepts)

- [Project Structure](#project-structure)

- [Writing Your First Test](#writing-your-first-test)

- [Common Patterns](#common-patterns)

- [Troubleshooting](#troubleshooting)

- [Real Examples](#real-examples)

  

---

  

## What is KUTTL?

  

**KUTTL** is a testing tool for Kubernetes operators that lets you:

- Test operator behavior without writing code

- Verify resources are created correctly

- Ensure status conditions are set properly

- Validate updates and deletions work as expected

  

Think of it as **"Given-When-Then" testing** but for Kubernetes:

- **Given**: Initial resources (create files)

- **When**: Wait for operator to reconcile

- **Then**: Assert expected state (assert files)

  

---

  

## Basic Concepts

  

### 1. Test Suites

A collection of test cases organized in directories.

  

```

test/kuttl/

├── kuttl-test.yaml # Test suite configuration

└── tests/

├── basic-test/ # Test case 1

└── update-test/ # Test case 2

```

  

### 2. Test Steps

Each test case contains numbered steps that run sequentially:

  

```

basic-test/

├── 00-setup.yaml # Step 0: Create initial resources

├── 01-assert.yaml # Step 1: Verify creation

├── 02-update.yaml # Step 2: Modify resources

├── 03-assert.yaml # Step 3: Verify update

└── 04-cleanup.yaml # Step 4: Delete resources

```

  

### 3. File Types

  

| File Type | Purpose | Example |

|-----------|---------|---------|

| `create-*.yaml` | Create resources | `00-create-pod.yaml` |

| `assert-*.yaml` | Verify resources exist with expected values | `01-assert-pod-ready.yaml` |

| `errors-*.yaml` | Verify resources DON'T exist | `02-errors-pod-deleted.yaml` |

| `cleanup-*.yaml` | Delete resources | `03-cleanup.yaml` |

  

### 4. How KUTTL Works

  

```

Step 00: Create Resource

↓

Wait for reconcile (default: 30s)

↓

Step 01: Assert Resource State

↓

Wait...

↓

Step 02: Update Resource

↓

Wait...

↓

Step 03: Assert Updated State

```

  

---

  

## Project Structure

  

### Recommended Layout

  

```

test/kuttl/

├── kuttl-test.yaml # Suite config

├── common/ # Reusable resources

│ ├── mock-objects/

│ │ ├── mock-resources.yaml

│ │ ├── assert-mock-objects-created.yaml

│ │ ├── cleanup-mock-objects.yaml

│ │ └── errors-mock-objects.yaml

│ └── my-operator-instance/

│ ├── create-instance.yaml

│ ├── assert-instance.yaml

│ ├── cleanup-instance.yaml

│ └── errors-instance.yaml

└── tests/

├── basic-configuration/

│ ├── 00-mock-resources.yaml -> ../../common/mock-objects/mock-resources.yaml

│ ├── 01-assert-mocks.yaml -> ../../common/mock-objects/assert-mock-objects-created.yaml

│ ├── 02-create-instance.yaml -> ../../common/my-operator-instance/create-instance.yaml

│ └── 03-assert-instance.yaml -> ../../common/my-operator-instance/assert-instance.yaml

└── update-configuration/

└── (similar structure)

```

  

**Why use symlinks?**

- ✅ Reuse common setup across tests

- ✅ Single source of truth

- ✅ Easy to maintain

  

---

  

## Writing Your First Test

  

### Step 1: Configure Test Suite

  

Create `kuttl-test.yaml`:

  

```yaml

apiVersion: kuttl.dev/v1beta1

kind: TestSuite

reportFormat: xml # Generate XML report

reportName: kuttl-report-my-operator # Report filename

reportGranularity: test # Report per test

namespace: my-operator-namespace # Test namespace

timeout: 600 # Timeout per step (seconds)

parallel: 1 # Run tests in sequence

suppress:

- events # Don't log k8s events

```

  

**Key Settings:**

- `timeout`: How long to wait for each step (increase for slow operations)

- `parallel`: Set to `1` to avoid resource conflicts

- `namespace`: Where to create test resources

  

### Step 2: Create Your First Test Case

  

Create directory: `test/kuttl/tests/basic-test/`

  

#### **File 1: Create Resource** (`00-create-pod.yaml`)

  

```yaml

---

apiVersion: v1

kind: Pod

metadata:

name: test-pod

namespace: my-operator-namespace

spec:

containers:

- name: nginx

image: nginx:latest

ports:

- containerPort: 80

```

  

**What this does:** Creates a pod named `test-pod`

  

#### **File 2: Assert Pod Exists** (`01-assert-pod-ready.yaml`)

  

```yaml

---

apiVersion: v1

kind: Pod

metadata:

name: test-pod

namespace: my-operator-namespace

status:

phase: Running # Assert pod is running

conditions:

- type: Ready

status: "True" # Assert pod is ready

```

  

**What this checks:**

- ✅ Pod named `test-pod` exists

- ✅ Pod is in `Running` phase

- ✅ Pod has `Ready` condition = `True`

  

#### **File 3: Delete Pod** (`02-cleanup.yaml`)

  

```yaml

---

apiVersion: v1

kind: Pod

metadata:

name: test-pod

namespace: my-operator-namespace

```

  

**What this does:** Deletes the pod

  

#### **File 4: Assert Pod Deleted** (`03-errors-pod-gone.yaml`)

  

```yaml

---

apiVersion: v1

kind: Pod

metadata:

name: test-pod

namespace: my-operator-namespace

```

  

**What this checks:**

- ✅ Pod named `test-pod` does NOT exist (errors file = expect absence)

  

### Step 3: Run Your Test

  

```bash

# Run all tests

make kuttl-test

  

# Run specific test

kubectl kuttl test --config kuttl-test.yaml test/kuttl/tests/basic-test

```

  

---

  

## Common Patterns

  

### Pattern 1: Testing Operator CRD Creation

  

#### Create Custom Resource

```yaml

# 00-create-myresource.yaml

---

apiVersion: myoperator.example.com/v1

kind: MyResource

metadata:

name: test-instance

namespace: my-namespace

spec:

replicas: 3

image: myapp:latest

```

  

#### Assert Resource and Status

```yaml

# 01-assert-myresource.yaml

---

apiVersion: myoperator.example.com/v1

kind: MyResource

metadata:

name: test-instance

namespace: my-namespace

spec:

replicas: 3 # Assert spec values

image: myapp:latest

status:

conditions: # Assert status conditions

- type: Ready

status: "True"

reason: Deployed

message: All replicas ready

observedGeneration: 1 # Assert generation

```

  

**Key Points:**

- ✅ Check both `spec` (desired state) and `status` (observed state)

- ✅ Verify conditions are set correctly

- ✅ Partial matches work (don't need ALL fields)

  

### Pattern 2: Testing Updates

  

#### Update Resource

```yaml

# 02-update-myresource.yaml

---

apiVersion: myoperator.example.com/v1

kind: MyResource

metadata:

name: test-instance

namespace: my-namespace

spec:

replicas: 5 # Changed from 3 to 5

image: myapp:v2.0 # Updated image

```

  

#### Assert Update Applied

```yaml

# 03-assert-update.yaml

---

apiVersion: myoperator.example.com/v1

kind: MyResource

metadata:

name: test-instance

spec:

replicas: 5 # Verify new value

image: myapp:v2.0

status:

observedGeneration: 2 # Generation should increment

conditions:

- type: Ready

status: "True"

```

  

### Pattern 3: Testing Dependent Resources

  

When your operator creates other resources (Deployments, Services, etc.):

  

```yaml

# 01-assert-created-deployment.yaml

---

apiVersion: apps/v1

kind: Deployment

metadata:

name: myresource-deployment

namespace: my-namespace

spec:

replicas: 3

status:

readyReplicas: 3 # All replicas ready

conditions:

- type: Available

status: "True"

---

apiVersion: v1

kind: Service

metadata:

name: myresource-service

namespace: my-namespace

spec:

ports:

- port: 80

targetPort: 8080

```

  

**Pro Tip:** Use `---` to separate multiple resources in one file

  

### Pattern 4: Partial Matching

  

You don't need to specify ALL fields:

  

```yaml

# This works! Only checks what you specify

---

apiVersion: myoperator.example.com/v1

kind: MyResource

metadata:

name: test-instance

status:

conditions: # Only checking conditions

- type: Ready # Only checking this one condition

status: "True"

# Don't care about other status fields

```

  

KUTTL does **subset matching**:

- ✅ Checks specified fields exist with correct values

- ✅ Ignores fields you don't mention

- ✅ Allows extra fields not in assertion

  

### Pattern 5: Using Symlinks for Reusable Steps

  

```bash

# Create reusable common files

test/kuttl/common/

└── myresource/

├── create-myresource.yaml

└── assert-myresource.yaml

  

# Link them in tests

cd test/kuttl/tests/test-case-1/

ln -s ../../common/myresource/create-myresource.yaml 00-create.yaml

ln -s ../../common/myresource/assert-myresource.yaml 01-assert.yaml

```

  

**Benefits:**

- ✅ One source of truth

- ✅ Easy updates (change once, affects all tests)

- ✅ Consistent across test cases

  

### Pattern 6: Testing Field Omission

  

Sometimes you want to verify a field is NOT present:

  

```yaml

# For optional fields with omitempty, they may not appear when empty

# DON'T assert them when they're empty

  

# ❌ WRONG - This will fail if field is omitted

status:

optionalField: "" # Field may not exist in JSON

  

# ✅ RIGHT - Don't check optional empty fields

status:

conditions:

- type: Ready

status: "True"

# Don't mention optionalField

```

  

**Or** use errors file to assert absence:

  

```yaml

# Check resource exists but specific field doesn't

# (This is advanced, usually just omit the check)

```

  

---

  

## Troubleshooting

  

### Problem 1: Test Times Out

  

**Error:**

```

test step failed after 600s

```

  

**Solutions:**

```yaml

# Increase timeout in kuttl-test.yaml

timeout: 1200 # 20 minutes

  

# Or check if resource is stuck

kubectl get pods -n my-namespace

kubectl describe pod <pod-name> -n my-namespace

```

  

### Problem 2: Assertion Mismatch

  

**Error:**

```

value mismatch, expected: foo != actual: bar

```

  

**Solutions:**

1. Check actual resource state:

```bash

kubectl get myresource test-instance -n my-namespace -o yaml

```

  

2. Compare with assertion file

3. Common issues:

- Field name typo

- Wrong value type (`"3"` vs `3`)

- Field omitted (use `omitempty`)

- Wrong condition order

  

### Problem 3: Field Is Missing

  

**Error:**

```

.status.myField: key is missing from map

```

  

**Solutions:**

  

**Option 1:** Field might be omitted (has `omitempty` tag)

```yaml

# Don't assert fields that might be omitted

# Remove the check from assertion file

```

  

**Option 2:** Field should exist but doesn't

```go

// Check operator code - is field being set?

instance.Status.MyField = "value"

```

  

**Option 3:** Wrong field name

```yaml

# Check CRD for actual JSON field name

kubectl get crd myresources.myoperator.com -o yaml | grep myField

```

  

### Problem 4: Unknown Field Warning

  

**Error:**

```

Warning: unknown field "spec.myField"

```

  

**Solutions:**

1. Field name is wrong - check CRD:

```bash

kubectl explain myresource.spec

```

  

2. Field was renamed - update test file

3. API version mismatch - check apiVersion

  

### Problem 5: Resource Already Exists

  

**Error:**

```

myresource "test-instance" already exists

```

  

**Solutions:**

1. Previous test didn't clean up:

```yaml

# Add cleanup step at end of previous test

# XX-cleanup.yaml

```

  

2. Namespace not cleaned between tests:

```bash

# Delete namespace manually

kubectl delete namespace my-namespace

```

  

3. Use unique names:

```yaml

metadata:

name: test-instance-{{ .TestName }}

```

  

### Problem 6: Condition Order Mismatch

  

**Error:**

```

slice length mismatch: expected 2 conditions, got 4

```

  

**Solution:**

KUTTL checks conditions **in order**. Your assertion must match the operator's order:

  

```yaml

# ❌ WRONG ORDER

conditions:

- type: Ready

- type: Deployed

- type: Available

  

# ✅ CORRECT - Match operator's order

conditions:

- type: Ready

- type: Available

- type: Deployed

```

  

**How to find correct order:**

```bash

kubectl get myresource test-instance -o jsonpath='{.status.conditions[*].type}'

```

  

### Problem 7: Test Cleanup Not Running

  

**Issue:** Cleanup steps don't run when assertion fails

  

**Solution:** KUTTL stops on first failure. Cleanup won't run if earlier step fails.

  

**Workaround:**

1. Fix failing test first

2. Or manually cleanup:

```bash

kubectl delete namespace my-namespace

```

  

---

  

## Real Examples from OpenStack Lightspeed Operator

  

### Example 1: Basic Instance Creation

  

**Test Case:** Create OpenStackLightspeed instance with OCP RAG disabled

  

**File Structure:**

```

basic-openstack-lightspeed-configuration/

├── 00-mock-resources.yaml # Create mock LLM server

├── 01-assert-mock-objects-created.yaml # Verify mocks ready

├── 02-create-instance.yaml # Create CR instance

├── 03-assert-instance.yaml # Verify instance + OLSConfig

├── 04-cleanup-instance.yaml # Delete instance

├── 05-errors-instance.yaml # Verify deleted

├── 06-cleanup-mocks.yaml # Delete mocks

└── 07-errors-mocks.yaml # Verify mocks deleted

```

  

**Step 0: Create Mocks**

```yaml

# 00-mock-resources.yaml

---

apiVersion: v1

kind: Secret

metadata:

name: openstack-lightspeed-apitoken

namespace: openshift-lightspeed

type: Opaque

stringData:

apitoken: fake-token-for-testing

---

apiVersion: v1

kind: Pod

metadata:

name: mock-llm-api-server-pod

namespace: openshift-lightspeed

spec:

serviceAccountName: default # Inherit pull secrets

containers:

- name: mock-llm-api-server

image: registry.redhat.io/ubi8/python-311:latest

command: ["/bin/bash", "-c"]

args:

- |

python3 /app/mock_server.py

ports:

- containerPort: 8000

volumeMounts:

- name: app-code

mountPath: /app

volumes:

- name: app-code

configMap:

name: mock-llm-code

```

  

**Step 2: Create Instance**

```yaml

# 02-create-instance.yaml

---

apiVersion: lightspeed.openstack.org/v1beta1

kind: OpenStackLightspeed

metadata:

name: openstack-lightspeed

namespace: openshift-lightspeed

spec:

llmEndpoint: http://mock-llm-api-server-pod:8000/v1

llmEndpointType: openai

llmCredentials: openstack-lightspeed-apitoken

modelName: ibm-granite/granite-3.1-8b-instruct

tlsCACertBundle: openstack-lightspeed-cert

llmProjectID: test-project-id

llmDeploymentName: test-deployment-name

llmAPIVersion: v1

enableOCPRAG: false # OCP RAG disabled

```

  

**Step 3: Assert Instance Created**

```yaml

# 03-assert-instance.yaml

---

# First: Check OLSConfig was created by operator

apiVersion: ols.openshift.io/v1alpha1

kind: OLSConfig

metadata:

name: cluster

spec:

llm:

providers:

- name: openstack-lightspeed-provider

type: openai

url: http://mock-llm-api-server-pod:8000/v1

credentialsSecretRef:

name: openstack-lightspeed-apitoken

models:

- name: ibm-granite/granite-3.1-8b-instruct

parameters:

maxTokensForResponse: 2048

ols:

rag:

- image: quay.io/openstack-lightspeed/rag-content:os-docs-2025.2

indexPath: /rag/vector_db/os_product_docs

# Note: No OCP RAG entry because enableOCPRAG: false

status:

overallStatus: Ready # OLSConfig must be ready

---

# Second: Check OpenStackLightspeed status

apiVersion: lightspeed.openstack.org/v1beta1

kind: OpenStackLightspeed

metadata:

name: openstack-lightspeed

namespace: openshift-lightspeed

status:

# Note: activeOCPRAGVersion not checked (omitted when empty)

conditions:

- type: Ready

status: "True"

reason: Ready

message: Setup complete

- type: OCPRAGReady

status: "True"

reason: Ready

message: OCP RAG is disabled

- type: OpenShiftLightspeedOperatorReady

status: "True"

reason: Ready

message: OpenShift Lightspeed operator is ready.

- type: OpenStackLightspeedReady

status: "True"

reason: Ready

message: OpenStack Lightspeed created

```

  

**Key Learnings:**

- ✅ Assert BOTH the CR and dependent resources (OLSConfig)

- ✅ Check conditions in correct order

- ✅ Don't check fields with `omitempty` when they're empty

- ✅ Verify meaningful messages in conditions

  

### Example 2: Testing Updates with OCP Version Override

  

**Test Case:** Update instance to enable OCP RAG with version override

  

**Step 4: Update Instance**

```yaml

# 04-update-instance.yaml

---

apiVersion: lightspeed.openstack.org/v1beta1

kind: OpenStackLightspeed

metadata:

name: openstack-lightspeed

namespace: openshift-lightspeed

spec:

llmEndpoint: http://mock-llm-api-server-pod-UPDATE:8000/v1

llmEndpointType: bam # Changed provider type

llmCredentials: openstack-lightspeed-apitoken-UPDATE

modelName: ibm-granite/granite-3.1-8b-instruct-UPDATE

enableOCPRAG: true # Enable OCP RAG

ocpVersionOverride: "4.16" # Force version 4.16

```

  

**Step 5: Assert OLSConfig Updated**

```yaml

# 05-assert-olsconfig-update.yaml

---

apiVersion: ols.openshift.io/v1alpha1

kind: OLSConfig

metadata:

name: cluster

spec:

llm:

providers:

- name: openstack-lightspeed-provider

type: bam # Verify type changed

url: http://mock-llm-api-server-pod-UPDATE:8000/v1

ols:

rag:

- image: quay.io/openstack-lightspeed/rag-content:os-docs-2025.2

indexPath: /rag/vector_db/os_product_docs

- image: quay.io/openstack-lightspeed/rag-content:os-docs-2025.2

indexID: ocp-product-docs-4_16 # OCP 4.16 RAG added!

indexPath: /rag/ocp_vector_db/ocp-4.16

```

  

**Step 6: Assert OpenStackLightspeed Status**

```yaml

# 06-assert-openstacklightspeed-update.yaml

---

apiVersion: lightspeed.openstack.org/v1beta1

kind: OpenStackLightspeed

metadata:

name: openstack-lightspeed

namespace: openshift-lightspeed

status:

activeOCPRAGVersion: "4.16" # Override value appears!

conditions:

- type: Ready

status: "True"

- type: OCPRAGReady

status: "True"

- type: OpenShiftLightspeedOperatorReady

status: "True"

- type: OpenStackLightspeedReady

status: "True"

```

  

**Key Learnings:**

- ✅ Split complex assertions (OLSConfig and CR status) into separate files

- ✅ Non-empty strings appear even with `omitempty`

- ✅ Verify override is applied (4.16 RAG, not auto-detected version)

- ✅ Check dependent resource updated correctly

  

---

  

## Best Practices

  

### ✅ DO

  

1. **Use descriptive names**

```yaml

# Good

00-create-mock-llm-server.yaml

01-assert-mock-llm-ready.yaml

  

# Bad

00-setup.yaml

01-check.yaml

```

  

2. **Test one thing per test case**

- `basic-configuration/` - Tests basic setup

- `update-configuration/` - Tests updates

- `deletion/` - Tests cleanup

  

3. **Use symlinks for reusable resources**

```bash

ln -s ../../common/mocks/mock-server.yaml 00-mocks.yaml

```

  

4. **Add comments explaining non-obvious checks**

```yaml

status:

activeOCPRAGVersion: "4.16" # Should use override, not auto-detected

```

  

5. **Check both spec and status**

```yaml

spec:

replicas: 3 # Desired state

status:

readyReplicas: 3 # Actual state

```

  

6. **Verify meaningful condition messages**

```yaml

conditions:

- type: Ready

status: "True"

message: Setup complete # Helpful for debugging

```

  

7. **Use proper timeout values**

```yaml

# For quick operations

timeout: 300 # 5 minutes

  

# For image pulls / slow operations

timeout: 1200 # 20 minutes

```

  

### ❌ DON'T

  

1. **Don't hardcode namespace everywhere**

```yaml

# Use kuttl-test.yaml namespace setting

namespace: openshift-lightspeed

```

  

2. **Don't check every field**

```yaml

# Only check what matters for the test

status:

conditions: # Just check conditions

- type: Ready

status: "True"

# Don't list every status field

```

  

3. **Don't ignore cleanup**

```yaml

# Always add cleanup steps

XX-cleanup.yaml

XX-errors.yaml

```

  

4. **Don't use absolute timing**

```yaml

# Bad: sleep for exact time

# Good: Let KUTTL poll with timeout

```

  

5. **Don't create multiple resources in one step without reason**

```bash

# Separate concerns

00-create-mocks.yaml # Mocks

02-create-instance.yaml # Instance

  

# Not: 00-create-everything.yaml

```

  

6. **Don't assert on fields with `omitempty` when empty**

```yaml

# If field has omitempty and is empty, it won't appear

# Don't check it!

```

  

---

  

## Quick Reference

  

### File Naming Convention

```

<step-number>-<action>-<resource>.yaml

  

Examples:

00-create-pod.yaml

01-assert-pod-ready.yaml

02-update-pod.yaml

03-assert-pod-updated.yaml

04-cleanup-pod.yaml

05-errors-pod-deleted.yaml

```

  

### Common Commands

  

```bash

# Run all tests

make kuttl-test

  

# Run specific test

kubectl kuttl test --config kuttl-test.yaml test/kuttl/tests/my-test

  

# Run with verbose output

kubectl kuttl test --config kuttl-test.yaml test/kuttl/tests/my-test --verbose

  

# Debug - show all resources

kubectl kuttl test --config kuttl-test.yaml test/kuttl/tests/my-test --verbose=5

  

# Check test results

cat kuttl-report-*.xml

```

  

### Useful kubectl Commands for Debugging

  

```bash

# Check if resource exists

kubectl get myresource -n my-namespace

  

# See full resource YAML

kubectl get myresource my-instance -n my-namespace -o yaml

  

# Check conditions

kubectl get myresource my-instance -n my-namespace -o jsonpath='{.status.conditions}'

  

# Watch resources

kubectl get pods -n my-namespace -w

  

# Check operator logs

kubectl logs -n my-operator-namespace deployment/my-operator-controller-manager -f

```

  

---

  

## Summary

  

**KUTTL Testing in 5 Steps:**

  

1. **Configure** - Create `kuttl-test.yaml` with timeout and namespace

2. **Create** - Write `XX-create-*.yaml` files to create resources

3. **Assert** - Write `XX-assert-*.yaml` files to verify expected state

4. **Cleanup** - Write `XX-cleanup-*.yaml` and `XX-errors-*.yaml` files

5. **Run** - Execute `make kuttl-test` and fix failures

  

**Remember:**

- ✅ Tests run sequentially (00, 01, 02...)

- ✅ Each step waits for timeout before failing

- ✅ Assertions do subset matching (only checks what you specify)

- ✅ `assert-*.yaml` = expect resource exists

- ✅ `errors-*.yaml` = expect resource doesn't exist

- ✅ Use symlinks to reuse common files

- ✅ Split complex assertions into separate files

  

---

  

## Resources

  

- **KUTTL Documentation**: https://kuttl.dev/

- **Examples**: Check `test/kuttl/tests/` in this repository

- **Kubernetes API Reference**: https://kubernetes.io/docs/reference/

- **Troubleshooting**: See [Troubleshooting](#troubleshooting) section above

  

---

  

**Happy Testing! 🎉**

  

If you have questions or improvements for this guide, please contribute!
