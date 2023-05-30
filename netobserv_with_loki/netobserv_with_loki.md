## Install loki-operator



## Install LokiStack

1. Create the Namespace:
```
kubectl create ns netobserv
```
2. Create an S3 bucket storage for Loki
```
S3_NAME="netobserv-loki"
AWS_REGION="eu-west-1"
AWS_KEY=$(aws configure get aws_access_key_id)
AWS_SECRET=$(aws configure get aws_secret_access_key)

aws s3api create-bucket --bucket $S3_NAME  --region $AWS_REGION --create-bucket-configuration LocationConstraint=$AWS_REGION
```
3. Create a secret with with AWS data
```
kubectl create -n netobserv secret generic lokistack-dev-s3 \
  --from-literal=bucketnames="$S3_NAME" \
  --from-literal=endpoint="https://s3.${AWS_REGION}.amazonaws.com" \
  --from-literal=access_key_id="${AWS_KEY}" \
  --from-literal=access_key_secret="${AWS_SECRET}" \
  --from-literal=region="${AWS_REGION}"
```
4. Create LokiStack
```
oc apply -f - <<EOF
heredoc> apiVersion: loki.grafana.com/v1
kind: LokiStack                 
metadata:                       
  name: loki                          
spec:                           
  tenants:                               
    mode: openshift-network
  managementState: Managed
  replicationFactor: 1
  storage:
    schemas:
      - effectiveDate: '2020-10-11'
        version: v11
    secret:
      name: lokistack-dev-s3
      type: s3
  size: 1x.extra-small
  storageClassName: gp2-csi
```
