# OCP Tips and Tricks
## Tips and Tricks for installing and using OCP 

**Add an htpasswd provider:**
https://docs.openshift.com/container-platform/4.4/authentication/identity_providers/configuring-htpasswd-identity-provider.html

This can also be done through the GUI
- Administration/Cluster Settings/Global Configuration/OAuth
- To add a new htpasswd file click “Add” 
- To delete an authenticator click on Yaml and delete the section from spec
- To delete all providers, click on Yaml and delete everything under spec and then add “ {}” on the spec line

First create the htpasswd file
```
htpasswd -c -B -b ./htpasswd rjkirk R0bertK1rk
htpasswd -B -b ./htpasswd admin P@ssw0rd
htpasswd -B -b ./htpasswd mflanner P@ssw0rd
```

Next create the secret
```
oc create secret generic htpass-secret --from-file=htpasswd=./htpasswd -n openshift-config
```
Next create the CR file:
```
vi htpasswd.cr
```
```
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: my_htpasswd_provider 
    mappingMethod: claim 
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
```
Apply the CR:
```
oc apply -f htpasswd.cr
```
Login:
```
oc login -u mflanner
```
Delete Users
```
oc get users
oc get identities
oc delete user mflanner
oc delete identity htpasswd:mflanner
```
Get all nodes
```
oc get nodes
```
Creating new nodes in GovCloud
1. Copy the user-data from an existing worker node
  - In EC2 Dashboard, Instances, right-click worker instance (m5.xlarge)
  - Instance Settings, View/Change User Data
  - Copy entire user data text field to clipboard and then click “cancel”
2. Create 3x new instances m5.4xlarge (Ensure the bastion instance is running)
  - In EC2 dashboard, click Launch Instance.
  - Choose “My AMIs” and select the RHCOS image you uploaded earlier
  - Select m5.4xlarge for Instance Type, Click Next: Configure Instance Details
  - “Number of instances” == 3
  - “Network” == Your OCP4 VPC (not the default VPC!)
  - “Subnet” == I’m lazy, so put all 3x instances in the same _private_ subnet
  - “IAM role” == Choose the worker profile that was created via Terraform earlier
    - This may be “none” in GovCloud
  - “User data” == paste the user-data we copied earlier
  - Click Next: Add Storage
  - Increase root volume == 120GB -- and delete on termination
  - Click Add a new volume for OCS == 500GB -- and delete on termination
  - Click Next: Add Tags
  - Click Add Tag:  KEY = kubernetes.io/cluster/YOURCLUSTERNAME   VALUE = shared
  - Click Add Tag (yes there is a dash at the end):  KEY = Name  VALUE = YOURCLUSTERNAME-storage-
  - Click Next: Configure Security Group
  - Security Group == existing Worker SG created previously via Terraform
  - Click Review and Launch
  - Key Pair == Proceed without a keypair (Ignition will deploy the key-pair defined during openshift-install)
3. Wait a while, and check/approve the CSR requests (see below)
  - SSH into bastion host
  - oc get csr
  - oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
    - This may need to be ran multiple times until all CSRs are shown as approved in “oc get csr” output
  - Realize that resistance is futile, and that more CSRs are created
  - Approve some more CSRs until oc get nodes shows the new nodes are Ready
  - Congratulations, you now have more OCP nodes.  Go to https://github.com/jaredhocutt/openshift4-guides/blob/master/docs/ocs_bare_metal.md




Create GPU nodes
1. Follow creating new nodes in gov cloud above but skip the storage parts
2. Entitle nodes: https://access.redhat.com/solutions/4908771
3. Install Nvidia GPU operators: https://docs.nvidia.com/datacenter/kubernetes/openshift-on-gpu-install-guide/index.html#openshift-gpu-support-install-via-operatorhub
4. https://cloud.garr.it/support/kb/kubernetes/insufficient_gpu_even_if_there_are/
5. https://cloud.garr.it/support/kb/kubernetes/insufficient_gpu_even_if_there_are/
6. create a new project gpu-operator-resources via gui
7. install nvidia gpu operator in this project
8. Once installed click on Node Feature Discovery and then click create instance
9. Click create
10. Go back to installed operators and click on Nvidia GPU operator and then click create instance
11. Click create


