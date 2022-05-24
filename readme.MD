# ConfigMaps & Secrets

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.svg)](https://shell.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https%3A%2F%2Fgithub.com%2Fnordineb%2Fklabs-configmaps-secrets)

> Application configuration for specific environments should not be bundled inside the app but rather made available by the environment where the workload is running.

👉 Configmaps & Secrets reside in a namespace and can only be referenced by Pods in that same namespace.

## Environment variables from a configmap

Goals:
- Import all values from a configmaps into env. vars.
- Only import a specific value from configMap into a env. var.
- Make configmaps available using volumes
- Understand kubernetes behavior when configmaps are updated  
```
kubectl create configmap single-config --from-literal=URL=https://google.com
kubectl create configmap multi-config --from-literal=APPSETTING1=foo-setting  --from-literal=APPSETTING2=bar-setting
kubectl describe cm/single-config
```

```
kubectl apply -f mynginx.yaml 
kubectl exec -it  mynginx --  env
kubectl exec -it  mynginx --  ls /etc/config/
kubectl exec -it  mynginx --  cat /etc/config/APPSETTING1
```

👉 Change the value of APPSETTING1 of the configmap multi-config and see if anything has changed in the pod
```
kubectl delete configmap multi-config
kubectl exec -it  mynginx --  ls /etc/config/
kubectl create configmap multi-config --from-literal=APPSETTING1=xxx --from-literal=yyy
kubectl exec -it  mynginx --  ls /etc/config/
```

👉 ConfigMaps injected as environment variables are not updated automatically and require a pod restart.

👉 Kubernetes auto-updates the config map into POD if mounted as a volume (unless `subpath` is used)

### subpath
```
kubectl apply -f volumes/nginx-deployment.yaml
```
- Get the public IP of the lb 
- check the web page
 
```
kubectl apply -f volumes/configmap-nginxconf.yaml 
```
👉 Notice how text can be embedded in a configmap

```
kubectl apply -f volumes/nginx-deployment-custom.yaml 
```
- Get the public IP of the lb 
- check the web page

👉 **subPath is used to replace a specific file that already exists on the filesystem, otherwise all content is hidden.**

## Environment variables from a secret
````
kubectl create secret generic prod-db-secret --from-literal=DB_PASSWORD=Pass@Word! --from-literal=APIKEY=123123123123
kubectl describe secret/prod-db-secret 
````

👉 Secrets are not encrypted 
```
kubectl get secret prod-db-secret -o jsonpath="{.data.DB_PASSWORD}" | base64 --decode 
kubectl get secret prod-db-secret -o jsonpath="{.data.APIKEY}" | base64 --decode 
```

```
kubectl apply -f secrets/mysecretnginx.yaml
kubectl exec -it  mysecretnginx --  env
kubectl exec -it  mysecretnginx --  ls /etc/secret-volume/
kubectl exec -it  mysecretnginx --  cat /etc/secret-volume/APIKEY
````

👉 Change the secrets and check if secrets have the same behavior than configmaps
```
kubectl delete secret prod-db-secret
kubectl create secret generic prod-db-secret --from-literal=DB_PASSWORD=XXXXXXXXXX --from-literal=APIKEY=YYYYYYYYYY
```

## external secrets

Create a SA with access to GSM, export the access key to a file
```
export PROJECT_ID=$(gcloud config get-value project)
gcloud iam service-accounts create external-secrets
gcloud secrets add-iam-policy-binding mysecret --member "serviceAccount:external-secrets@$PROJECT_ID.iam.gserviceaccount.com" --role "roles/secretmanager.secretAccessor"
gcloud iam service-accounts keys create key.json --iam-account=external-secrets@$PROJECT_ID.iam.gserviceaccount.com
```

Install external secret operator, the SA key must be stored as akubernetes secret
```
kubectl create ns eso
kubectl config set-context --current --namespace=eso
kubectl create secret --namespace eso generic gcpsm-secret --from-file=secret-access-credentials=key.json
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n eso --set installCRDs=true
sleep 30
```

Create a secret in GSM
```
echo -ne '{"password":"ABCABC123!!!}' | gcloud secrets create mysecret --data-file=-
```

Create a link to the GSM secret
```
export PROJECT_ID=$(gcloud config get-value project)
cat <<EOF | kubectl apply -f - 
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-backend
spec:
  provider:
      gcpsm:                                                  
        auth:
          secretRef:
            secretAccessKeySecretRef:
              name: gcpsm-secret                        
              key: secret-access-credentials
              namespace: eso                      
        projectID: $PROJECT_ID
EOF
```

Create an external secret 
```
ApiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: gcp-external-secret
  namespace: test
spec:
  secretStoreRef:
    kind: SecretStore
    name: gcp-backend          
  target:                                                             
    name: secret-to-be-created 
  data:                                                       
  - secretKey: password_file 
    remoteRef:            
      key: mysecret
EOF
```

👉 Verify that a kubernetes secret has been created by ESO 🎉🎉🎉