# eks-appmesh
Setting up AppMesh on AWS EKS

## Steps
### Create EKS cluster
```shell
eksctl create cluster \
    --name kamil-appmesh \
    --region us-east-1 \
    --managed \
    --version 1.18 \
    --ssh-access \
    --external-dns-access \
    --asg-access \
    --full-ecr-access \
    --appmesh-access \
    --alb-ingress-access \
    --node-type m5.large \
    --node-private-networking \
    --vpc-private-subnets=subnet-5768e71f,subnet-c8935b92 \
    --vpc-public-subnets=subnet-7b64eb33,subnet-66935b3c
```

### Extend the Instance profile
:warning: The recommended `nodegroup` filter doesn't work for managed worker node groups
```shell
INSTANCE_PROFILE_NAME=$(aws iam list-instance-profiles | jq -r '.InstanceProfiles[].InstanceProfileName' | grep eks)

ROLE_NAME=$(aws iam get-instance-profile --instance-profile-name $INSTANCE_PROFILE_NAME | jq -r '.InstanceProfile.Roles[] | .RoleName')

echo $ROLE_NAME

aws iam put-role-policy --role-name $ROLE_NAME --policy-name AppMesh-Policy-For-Worker --policy-document file://k8s-appmesh-worker-policy.json

# test the access to appmesh
kubectl apply -f awscli.yaml
kubectl get jobs
kubectl logs jobs/awscli
```

### Deploy Yelb app
```shell
cd aws-app-mesh-examples/walkthroughs/eks-getting-started/
kubectl apply -f infrastructure/yelb_initial_deployment.yaml

# verify
kubectl get all -n yelb

# get the external loadbalancer URL and open it in a browser
kubectl get service yelb-ui -n yelb
```

### Install AppMesh Injector Controller
```shell
helm repo list | grep -q eks-charts || helm repo add eks https://aws.github.io/eks-charts
kubectl create ns appmesh-system
helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system

# verify
kubectl get pods -n appmesh-system

# enable automatic injection of a sidecar container
kubectl label namespace yelb mesh=yelb
kubectl label namespace yelb appmesh.k8s.aws/sidecarInjectorWebhook=enabled

# create the appmesh
kubectl apply -f - <<EOF
apiVersion: appmesh.k8s.aws/v1beta2
kind: Mesh
metadata:
  name: yelb
spec:
  namespaceSelector:
    matchLabels:
      mesh: yelb
EOF

# check the appmesh exists
kubectl apply -f awscli.yaml
kubectl wait --for=condition=complete --timeout=30s job/awscli
kubectl logs jobs/awscli
```

### Create AppMesh components for Yelb
```shell
kubectl apply -f infrastructure/appmesh_templates/appmesh-yelb-redis.yaml
kubectl apply -f infrastructure/appmesh_templates/appmesh-yelb-db.yaml
kubectl apply -f infrastructure/appmesh_templates/appmesh-yelb-appserver.yaml
kubectl apply -f infrastructure/appmesh_templates/appmesh-yelb-ui.yaml

```

## Resources
* https://aws.amazon.com/blogs/containers/getting-started-with-app-mesh-and-eks/
* :warning: Broken on K8s 1.18+ https://aws.amazon.com/blogs/compute/learning-aws-app-mesh/
