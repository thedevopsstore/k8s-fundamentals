## RBAC < ROle based Access control

#### Mke sure Cluster is create and OIDC Provider exists use the below example to create cluster

https://github.com/thedevopsstore/k8s-fundamentals.git

cd k8s-fundamentals/02-K8s-create-cluster/eksctl/07-OIDC-provider

eksctl create cluster -f cluster.yaml

### Create pod

kubectl create deploy nginx --image=nginx


### Create AWS IAM User

```
aws iam create-user --user-name dev
aws iam create-access-key --user-name dev | tee /tmp/create_output.json
```
#### To make it easy to switch back and forth between the admin user you created the cluster with, and this new dev, run the following command to create a script that when sourced, sets the active user to be dev:
```
cat << EoF > dev_creds.sh
export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/create_output.json)
export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
EoF
```


### MAP AN IAM USER TO K8S

```

# Next, we’ll define a k8s user called rbac-user, and map to its IAM user counterpart. Run the following to get the existing ConfigMap and save into a file called aws-auth.yaml:

kubectl get configmap -n kube-system aws-auth -o yaml | grep -v "creationTimestamp\|resourceVersion\|selfLink\|uid" | sed '/^  annotations:/,+2 d' > aws-auth.yaml


# Next append the rbac-user mapping to the existing configMap < update the account ID >

cat << EoF >> aws-auth.yaml
data:
  mapUsers: |
    - userarn: arn:aws:iam::${ACCOUNT_ID}:user/dev
      username: rbac-user
EoF



# Some of the values may be dynamically populated when the file is created. To verify everything populated and was created correctly, run the following:

cat aws-auth.yaml

# Next, apply the ConfigMap to apply this mapping to the system:

kubectl apply -f aws-auth.yaml

```


#### Issue the following command to source the rbac-user’s AWS IAM user environmental variables:

```
source dev_creds.sh

aws eks update-kubeconfig --region us-east-1 --name eks-cluster



```

##### By running the above command, you’ve now set AWS environmental variables which should override the default admin user or role. To verify we’ve overrode the default user settings, run the following command:

```
aws sts get-caller-identity


```


# Now that we’re the admin user again, we’ll create a role called pod-reader that provides list, get, and watch access for pods and deployments, but only for the rbac-test namespace. Run the following to create this role:


cat << EoF > rbacuser-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["list","get","watch"]
- apiGroups: ["extensions","apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
EoF


cat << EoF > rbacuser-role-binding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EoF



cat << EoF >> aws-auth.yaml
data:
  mapRoles: |
    - userarn: arn:aws:iam::${ACCOUNT_ID}:user/dev
      username: rbac-user
EoF