# Thanos

## How to add Thanos rules?

1. Create a yaml file with the rules you want to add. In the example we'll name it thanos-rules.yml. It will be reused in the next step.
NOTE: The file needs to end with *.yml otherwise it will not be picked up by the thanos rules.

```yaml
groups:
  - name: test-thanos-alertmanager
    rules:
      - alert: TestAlertForThanos
        expr: vector(1)
        labels:
          severity: warning
        annotations:
          summary: Test alert
          description: This is a test alert to see if alert manager is working properly with Thanos
```

2. Overwrite the existing thanos-ruler-configmap in the infrastructure namespace with the new yaml file.

```bash
kubectl create configmap thanos-ruler-config --from-file=thanos-rules.yml -n infrastructure --dry-run=client -o yaml | kubectl replace -f -
```
