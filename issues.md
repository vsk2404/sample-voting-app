##### **ERROR:** unauthorized: {"errors":[{"code":"UNAUTHORIZED","message":"authentication required, visit https://aka.ms/acr/authorization for more information. CorrelationId: cd35934f-9540-46ae-8d87-28fc3351ea9c"}]}Error: Process completed with exit code 1.
> **Common Causes:**
> - No authentication configured: You’re trying to access a private ACR without logging in.
> - Invalid or expired credentials: You provided credentials, but they’re incorrect or expired.
> - GitHub Actions or CI/CD environment missing secrets: If this is from a pipeline, the credentials may not be set up in your secrets.
> **How to Get ACR Username and Password in Azure Portal**
> - 🧭 Go to the Azure Portal → https://portal.azure.com
> - 🔍 Navigate to your ACR: In the search bar, type “Container Registries” and select your registry.
> - 🛠️ Go to Access Keys: In the left-hand menu, select Access keys under the Settings section.
> - 🔓 Enable Admin User (if not already enabled): Toggle the switch to enable Admin user.
> - 👤 Copy Username and Passwords
> - You’ll see:
> - Username (usually the registry name)
> - Password1 and Password2 (two interchangeable passwords)
> - You can click the 👁️ icon to view or 📋 icon to copy them.
> **Important Notes**
> - Admin user access is convenient for development/testing but is not recommended for production due to security concerns.
> - For secure automation (e.g., GitHub Actions), consider using an Azure service principal or federated identity (OIDC) instead.

##### **ERROR:** {"code":"InvalidTemplateDeployment","details":[{"code":"AvailabilityZoneNotSupported","message":"Preflight validation check for resource(s) for container service azuredevops in resource group azurecicd failed. Message: The zone(s) '2' for resource 'agentpool' is not supported. The supported zones for location 'westus2' are ''. Details: "}],"message":"The template deployment 'microsoft.aks-1746796838148' is not valid according to the validation procedure. The tracking id is '24a853bc-15d3-4c2e-8d5f-d103046904b1'. See inner errors for details."} template with region-safe defaults to avoid this issue?
> To get the AZURE_CREDENTIALS needed for GitHub Actions (azure/login or azure/aks-set-context), you must create a Service Principal (SP) in Azure and export its credentials in a specific JSON format.
> You can't get this directly from the Azure Portal, but you can generate it via the Azure CLI, which uses the portal's permissions under the hood.
> Steps to Generate AZURE_CREDENTIALS
> - 1. Open Azure Cloud Shell (or use local Azure CLI): In the Azure Portal, click on the Cloud Shell icon (top right). Use Bash or PowerShell.
> - 2. Run this command:
```
az ad sp create-for-rbac --name "github-actions-aks" --role contributor --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP_NAME> --sdk-auth
```
> Replace:
> - <SUBSCRIPTION_ID> – get it with: az account show --query id -o tsv
> - <RESOURCE_GROUP_NAME> – e.g., voting-app-rg
> - 3. Output: JSON Credentials
```
{
  "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientSecret": "your-secret",
  "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  ...
}
```
> 🔐 Add It to GitHub as a Secret
> - Go to your GitHub repository.
> - Click Settings > Secrets and variables > Actions.
> - Click New secret.
> - Name it AZURE_CREDENTIALS.
> - Paste the full JSON output from the CLI.

##### **ERROR:**
```
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  2m49s                default-scheduler  Successfully assigned default/vote-58dfcdf847-hxjcg to aks-agentpool-20350788-vmss000000
  Normal   Pulling    83s (x4 over 2m48s)  kubelet            Pulling image "venkat.azurecr.io/vote-app-image:vote-77"
  Warning  Failed     82s (x4 over 2m45s)  kubelet            Failed to pull image "venkat.azurecr.io/vote-app-image:vote-77": failed to pull and unpack image "venkat.azurecr.io/vote-app-image:vote-77": failed to resolve reference "venkat.azurecr.io/vote-app-image:vote-77": failed to authorize: failed to fetch anonymous token: unexpected status from GET request to https://venkat.azurecr.io/oauth2/token?scope=repository%3Avote-app-image%3Apull&service=venkat.azurecr.io: 401 Unauthorized
  Warning  Failed     82s (x4 over 2m45s)  kubelet            Error: ErrImagePull
  Warning  Failed     51s (x6 over 2m45s)  kubelet            Error: ImagePullBackOff
  Normal   BackOff    40s (x7 over 2m45s)  kubelet            Back-off pulling image "venkat.azurecr.io/vote-app-image:vote-77"
  ```
> Resolution:
> - Attach ACR to AKS
> - Use this command to grant AKS permission to pull from ACR:
```
az aks update \
  --name <AKS_CLUSTER_NAME> \
  --resource-group <RESOURCE_GROUP_NAME> \
  --attach-acr <ACR_NAME>
```

##### **ERROR:**
```
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  43s                default-scheduler  Successfully assigned default/vote-7f6c447c45-c9rsz to aks-agentpool-20350788-vmss000000
  Normal   Pulling    43s                kubelet            Pulling image "venkat.azurecr.io/vote-app-image:vote-79"
  Normal   Pulled     34s                kubelet            Successfully pulled image "venkat.azurecr.io/vote-app-image:vote-79" in 8.707s (8.707s including waiting). Image size: 57248307 bytes.
  Normal   Created    18s (x3 over 34s)  kubelet            Created container: vote
  Normal   Started    18s (x3 over 34s)  kubelet            Started container vote
  Normal   Pulled     18s (x2 over 33s)  kubelet            Container image "venkat.azurecr.io/vote-app-image:vote-79" already present on machine
  Warning  BackOff    4s (x4 over 32s)   kubelet            Back-off restarting failed container vote in pod vote-7f6c447c45-c9rsz_default(a87422e3-f52f-468a-9225-9a101ac19a99)
  ```
> It means the container is crashing on startup. Kubernetes tries to start it, but it fails repeatedly.
> - Next Step: Check the Logs
> - Run this command to see why the container is crashing:
```
kubectl logs vote-7f6c447c45-c9rsz
```

##### kubectl logs vote-7f6c447c45-c9rsz
##### exec /usr/local/bin/gunicorn: exec format error
> clearly indicates a binary execution issue, which typically means:
> - Your Docker image was built for the wrong architecture.
> You're probably building your image on GitHub Actions' default runner (ubuntu-latest, which is x86_64) but your base image (or build setup) is for a different architecture — likely ARM64. This happens if:
> - You use a multi-arch base image (e.g., Alpine/ARM variant).
> - Your Dockerfile is copied from a non-x86 system (e.g., Raspberry Pi).
> - Or gunicorn inside the image is built for the wrong architecture.
> Check AKS Node Architecture
> - To confirm your AKS node architecture (should be amd64), run:
```
kubectl get nodes -o wide
kubectl describe node <node-name> | grep Architecture
```
> **Solution:** Build x86 Image Explicitly in GitHub Actions
```
- name: Build Docker image
  uses: docker/build-push-action@v5
  with:
    context: ./vote
    file: vote/Dockerfile
    platforms: linux/amd64   # 👈 Force x86_64 architecture
    push: false
    load: true
    tags: vote-app-image:temp
```
> Then tag and push as before. platforms: linux/amd64 ensures it's built for what AKS nodes expect (x86_64).

##### **ERROR**
```
ERROR: failed to solve: process "/bin/sh -c apt-get update &&     apt-get install -y --no-install-recommends curl &&     rm -rf /var/lib/apt/lists/*" did not complete successfully: exit code: 255
Error: buildx failed with: ERROR: failed to solve: process "/bin/sh -c apt-get update &&     apt-get install -y --no-install-recommends curl &&     rm -rf /var/lib/apt/lists/*" did not complete successfully: exit code: 255
```
> > usually occurs when building ARM64 Docker images on a GitHub Actions runner without proper emulation support.
> You're now building for linux/arm64, but GitHub Actions (which runs on linux/amd64) needs QEMU emulation to build ARM64 images.
```
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3
```

##### kubectl get pods
##### Unable to connect to the server: dial tcp: lookup azuredevops-dns-hfzqqvfc.hcp.westus2.azmk8s.io on 168.63.129.16:53: no such host
> means that your kubectl client can't resolve the DNS name for your AKS cluster's API server. This usually happens due to one of the following:
> - 1. Cluster Was Deleted
> - 2. Your kubectl Context is Outdated
> - 3. DNS Issues or Network Connectivity
> - 4. VPN or Firewall Rules Blocking Access