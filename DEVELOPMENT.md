# Development

## Setting up a local Kubernetes cluster

These instructions assume Ubuntu 20.04.

Start a persistent sudo session
```
sudo -s
```

Set up a kind cluster with a registry
```
./kind_with_registry.sh
```

Install kubectl
```
snap install kubectl --classic
```

## Local development

1. Make changes to code
2. Run the build script to build the container and push it to the registry in our local kind cluster.
```
./build.sh
```
3. Install into the local kind cluster by using the local registry rather than Docker Hub.
```
helm upgrade \
  --install \
  snowflake-prometheus-exporter \
  --set image.repository=localhost:5000/snowflake-prometheus-exporter \
  --set snowflake.account=$SNOWFLAKE_ACCOUNT \
  --set snowflake.username=$SNOWFLAKE_USERNAME \
  --set snowflake.password=$SNOWFLAKE_PASSWORD \
  ./helm/snowflake-prometheus-exporter
```
4. Check the pod is running
```
kubectl get pods
```
5. Expose the exporter locally on port 3000
```
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=snowflake-prometheus-exporter,app.kubernetes.io/instance=snowflake-prometheus-exporter" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace default port-forward $POD_NAME 3000:3000
```
6. Fetch the latest metrics
```
curl http://localhost:3000/metrics
```

## Local helm hosting

We will sign our helm chart before we use it. For this we will need a PGP key.

### Creating a PGP key
Create a new key. Remember the name you used when prompted.
```
gpg --gen-key
```

Export keys to a `.gpg`
```
gpg --export-secret-keys >~/.gnupg/secring.gpg
```

Sign the helm package
```
helm package --sign --key '<thenamefromkeyabove>' --keyring ~/.gnupg/secring.gpg --destination ./charts ./helm/snowflake-prometheus-exporter/
```

Verify the package
```
helm verify ./charts/snowflake-prometheus-exporter-1.0.0.tgz --keyring ~/.gnupg/secring.gpg
```

Update the local chart repo `index.yaml`
```
helm repo index ./charts
```

