

eksctl create cluster \
  --name my-cluster \
  --region ap-northeast-2 \
  --version 1.30 \
  --nodegroup-name my-managed-ng \
  --node-type t3.medium \
  --nodes 1 \
  --nodes-min 1 \
  --nodes-max 1 \
  --managed

  eksctl utils associate-iam-oidc-provider --region=ap-northeast-2 --cluster=my-cluster --approve


export cluster_name=my-cluster
export role_name=AmazonEKS_EFS_CSI_DriverRole
eksctl create iamserviceaccount \
    --name efs-csi-controller-sa \
    --namespace kube-system \
    --cluster $cluster_name \
    --role-name $role_name \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy \
    --approve \
    --region ap-northeast-2

TRUST_POLICY=$(aws iam get-role --role-name $role_name --query 'Role.AssumeRolePolicyDocument' | \
    sed -e 's/efs-csi-controller-sa/efs-csi-*/' -e 's/StringEquals/StringLike/')
aws iam update-assume-role-policy --role-name $role_name --policy-document "$TRUST_POLICY"

# EFS 설치
sed -i "s/<file-system-id>/$file_system_id/" 1_efs_pv.yaml

kubectl apply -f 1_efs_pv.yaml
kubectl apply -f 2_efs_pvc.yaml
kubectl apply -f 3_efs_deployment.yaml

kubectl describe pod efs-app

kubectl exec -it efs-app -- /bin/sh

df -h
mount | grep /data
127.0.0.1:/ on /data type nfs4 (rw,relatime,vers=4.1,rsize=1048576,wsize=1048576,namlen=255,hard,noresvport,proto=tcp,port=20376,timeo=600,retrans=2,sec=sys,clientaddr=127.0.0.1,local_lock=none,addr=127.0.0.1)

cd /data
echo "EFS Test" > test-efs.txt