# Use AWS Secrets Manager secrets in Kubernetes

It is possible to automatically mount and use secrets from AWS Secrets Manager in EKS workloads. This documentation page will show you how.
Note that this feature is optional for our EKS platform, and it's disabled by default. If you'd like to enable it for your cluster(s) please reach out to us.

When enabled, two extra components will be deployed on the EKS cluster: the [Secrets Store CSI driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver) and the [AWS Secrets Manager provider](https://github.com/aws/secrets-store-csi-driver-provider-aws). The CSI driver provides the in-cluster functionality to be able to mount secrets from external providers into Pods via a [Container Storage Interface (CSI)](https://kubernetes-csi.github.io/docs/) volume, and the AWS Secrets Manager provider integrates it with the AWS service.

Note that besides mounting secrets into the container's file system, it's also possible to sync them to actual Kubernetes `Secret` objects and use them as environment variables in Pods.

## Usage

To be able to mount a secret from AWS Secrets Manager, you first need to create a `SecretProviderClass` in the namespace you're going to be using it:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "arn:aws:secretsmanager:eu-west-1:[111122223333]:secret:MySecret-00AACC"
        jmesPath:
          - path: user.username
            objectAlias: dbusername
          - path: user.password
            objectAlias: dbpassword
      - objectName: "arn:aws:secretsmanager:eu-west-1:[111122223333]:secret:MySecret2-00AABB"
        objectAlias: foobar
```

You can specify multiple secrets in the same `SecretProviderClass`, and all of them will be mounted in the same directory when referenced in the Pod. For secrets in JSON format, you can use the `jmesPath` attribute to filter which attribtues from the JSON object to map. Provided the following JSON object, the example `SecretProviderClass` above would result in two files, `dbusername` and `dbpassword` with the respective contents `john.doe` and `123456`, plus the file `foobar` for the secret `MySecret2-00AABB`, which will contain the whole secret value, regardless of its format.

```json
{
  "user": {
    "username": "john.doe",
    "password": "password"
  }
}
```

Then the `SecretProviderClass` needs to be referenced from the Pod as a volume, like:

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-secrets
spec:
  containers:
    - name: busybox
      image: k8s.gcr.io/e2e-test-images/busybox:1.29
      command:
        - "/bin/sleep"
        - "10000"
      volumeMounts:
        - name: secrets
          mountPath: "/mnt/secrets-store"
          readOnly: true
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: aws-secrets
```

**Note** that the Pod needs have the propper permissions ([via IRSA](README.md#iam-roles))) to access the referenced secrets from Secrets Manager.

---

To be able to use a secret value as an environment variable, you need to specify the `secretObjects` attribute in the `SecretProviderClass` object, like:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
  secretObjects:
    - secretName: foosecret
      type: Opaque
      data:
        - objectName: foobar # name of the mounted content to sync. this could be the object name or object alias 
          key: username
  parameters:
    objects: |
      - objectName: "arn:aws:secretsmanager:eu-west-1:[111122223333]:secret:MySecret-00AACC"
        jmesPath:
          - path: user.username
            objectAlias: dbusername
          - path: user.password
            objectAlias: dbpassword
      - objectName: "arn:aws:secretsmanager:eu-west-1:[111122223333]:secret:MySecret2-00AABB"
        objectAlias: foobar
```

This `SecretProviderClass` will instruct the CSI driver to create and sync the specified `Secret` objects with the contents of the referenced mounted secrets. To be able to use the `Secret` you still need to define the `volumes` and `volumeMounts` in the Pod specification.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-secrets
spec:
  containers:
    - name: busybox
      image: k8s.gcr.io/e2e-test-images/busybox:1.29
      command:
        - "/bin/sleep"
        - "10000"
      volumeMounts:
        - name: secrets
          mountPath: "/mnt/secrets-store"
          readOnly: true
      env:
        - name: SECRET_USERNAME
          valueFrom:
            secretKeyRef:
              name: foosecret
              key: username
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: aws-secrets
```

Note that the synced `Secret` object will only exist as long as there's a Pod mounting the `SecretProviderClass`. If no Pods are using the `SecretProviderClass`, the CSI driver will stop syncing the `Secret` object and delete it.

## Documentation references

You can read more on how the Secrets Store CSI driver works in the [official documentation page](https://secrets-store-csi-driver.sigs.k8s.io/getting-started/usage.html). Similar with the AWS Secrets Manager provider, there's some more information in the [official AWS documentation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html).
