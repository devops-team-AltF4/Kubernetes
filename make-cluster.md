eksctl create cluster \
--name project4-staging \
--version 1.21 \
--region ap-northeast-2 \
--nodegroup-name staging-nodes \
--node-type t3.large \
--nodes 1
// --ssh-access
// --ssh-public-key=jae.pem

//delete
eksctl delete cluster --name=project4-staging


eksctl create cluster \
--name cluster-1 \
--version 1.21 \
--region us-west-2 \
--nodegroup-name c1-nodes \
--node-type t3.medium \
--nodes 2


eksctl create cluster -f cluster.yaml