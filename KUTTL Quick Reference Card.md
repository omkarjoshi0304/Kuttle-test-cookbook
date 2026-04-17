
## Test File Templates

  

### Create Resource

```yaml

# XX-create-myresource.yaml

---

apiVersion: myapi.example.com/v1

kind: MyResource

metadata:

name: test-instance

namespace: my-namespace

spec:

field1: value1

field2: value2

```

  

### Assert Resource Exists

```yaml

# XX-assert-myresource.yaml

---

apiVersion: myapi.example.com/v1

kind: MyResource

metadata:

name: test-instance

namespace: my-namespace

spec:

field1: value1 # Check spec values

status:

conditions: # Check status

- type: Ready

status: "True"

reason: Deployed

message: All ready

```

  

### Delete Resource

```yaml

# XX-cleanup.yaml

---

apiVersion: myapi.example.com/v1

kind: MyResource

metadata:

name: test-instance

namespace: my-namespace

```

  

### Assert Resource Deleted

```yaml

# XX-errors-deleted.yaml

---

apiVersion: myapi.example.com/v1

kind: MyResource

metadata:

name: test-instance

namespace: my-namespace

```

  

## Common Patterns

  

### Multiple Resources in One File

```yaml

---

apiVersion: v1

kind: Secret

metadata:

name: my-secret

---

apiVersion: v1

kind: ConfigMap

metadata:

name: my-config

```

  

### Partial Matching

```yaml

# Only checks specified fields

status:

conditions:

- type: Ready # Only this condition

status: "True"

# Other fields ignored

```

  

### Don't Check Empty omitempty Fields

```yaml

# ❌ DON'T - Field omitted when empty

status:

optionalField: ""

  

# ✅ DO - Skip optional empty fields

status:

conditions:

- type: Ready

```

  

## Test Suite Config

  

```yaml

# kuttl-test.yaml

apiVersion: kuttl.dev/v1beta1

kind: TestSuite

reportFormat: xml

reportName: kuttl-report

namespace: my-namespace

timeout: 600 # seconds per step

parallel: 1 # sequential

suppress:

- events

```

  

## File Naming

  

```

00-create-resource.yaml # Create

01-assert-ready.yaml # Verify

02-update-resource.yaml # Modify

03-assert-updated.yaml # Verify update

04-cleanup.yaml # Delete

05-errors-deleted.yaml # Verify deletion

```

  

## Common Commands

  

```bash

# Run all tests

make kuttl-test

  

# Run one test

kubectl kuttl test --config kuttl-test.yaml test/kuttl/tests/my-test

  

# Verbose mode

kubectl kuttl test --config kuttl-test.yaml test/kuttl/tests/my-test --verbose

  

# Check actual resource

kubectl get myresource test-instance -n my-namespace -o yaml

  

# Check conditions

kubectl get myresource test-instance -o jsonpath='{.status.conditions}'

  

# View logs

kubectl logs -n operator-namespace deployment/controller-manager -f

```

  

## Debugging Tips

  

### Test Fails - "key is missing from map"

- Field might have `omitempty` and is empty

- **Fix**: Don't assert on optional empty fields

  

### Test Fails - "value mismatch"

- Check actual value: `kubectl get resource -o yaml`

- Common issues: wrong type (`"3"` vs `3`), typo

  

### Test Fails - "slice length mismatch"

- Condition order matters!

- **Fix**: Match operator's condition order

  

### Test Times Out

- Increase `timeout` in kuttl-test.yaml

- Check if resource stuck: `kubectl describe`

  

### Unknown Field Warning

- Field name is wrong

- **Fix**: Check CRD: `kubectl explain myresource.spec`

  

## Symlink Commands

  

```bash

# Create symlink

ln -s ../../common/resource.yaml 00-resource.yaml

  

# List symlinks

ls -la *.yaml

  

# Remove symlink

rm 00-resource.yaml

```

  

## Conditions Check Template

  

```yaml

status:

conditions:

- type: Ready # Always first

status: "True"

reason: Ready

message: Setup complete

- type: MyCondition

status: "True" # or "False" or "Unknown"

reason: MyReason # CamelCase

message: Human readable

```

  

## Troubleshooting Checklist

  

- [ ] Is timeout long enough?

- [ ] Are conditions in correct order?

- [ ] Am I checking optional empty fields?

- [ ] Are field names correct? (check CRD)

- [ ] Is namespace correct?

- [ ] Did previous test cleanup?

- [ ] Are types correct? (string vs int)

- [ ] Did I run `make generate manifests`?

  

## Remember

  

✅ Tests run in numerical order (00, 01, 02...)

✅ Assertion = subset match (only checks what you specify)

✅ Use `assert-*.yaml` to expect presence

✅ Use `errors-*.yaml` to expect absence

✅ Increase timeout for slow operations

✅ Use symlinks to reuse common files

✅ Check BOTH spec and status

✅ Don't assert on `omitempty` empty fields

  

## Need More Help?

  

See full guide: `docs/KUTTL_TESTING_GUIDE.md`
