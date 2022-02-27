---
title: Kubernetes のノードプールについて
date: 2022-02-18 12:00
tags:
  - Containers
  - Azure Kubernetes Service (AKS)
---

こんにちは、Azure テクニカル サポート チームです。
今回は、よくお問い合わせをいただく AKS (Kubernetes) のノードプールについて、2021 年 7 月 12 日に米国の Core Infrastructure and Security Blog で公開された [Kubernetes Nodepools Explained](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/kubernetes-nodepools-explained/ba-p/2531581) を意訳して紹介いたします。

<!-- more -->

## はじめに
この記事では Kubernetes においてノードプールが使用されるユースケースについて、紹介いたします。

1. ノードプールとは何か？
1. システムノードプール、ユーザーノードプールとは？
1. Label と nodeSelector を使用して、どのようにアプリケーションの Pod がノードプールにスケジュールされるのか？
1. Taint と Tolerations を使用することで、どのように特定のアプリケーションだけがスケジュールされるのか？
1. クラスターのアップグレードに伴うリスクを低減するため、どのようにノードプールを使用するか？
1. 各ノードプールの自動スケールをどのように設定するか？

Kubernetes のクラスターでは、コンテナは Pod として Worker Node と呼ばれる VM へデプロイされます。

In a Kubernetes cluster, the containers are deployed as pods into VMs called worker nodes.

These nodes are identical as they use the same VM size or SKU.

This was just fine until we realized we might need nodes with different SKU for the following reasons:

 

Prefer to deploy Kubernetes system pods (like CoreDNS, metrics-server, Gatekeeper addon) and application pods on different dedicated nodes. This is to prevent misconfigured or rogue application pods to accidentally killing system pods.
Some pods requires either CPU or Memory intensive and optimized VMs.
Some pods are processing ML/AI algorithms and needs GPU enabled VMs. These GPU enabled VMs should be used only by certain pods as they are expensive.
Some pods/jobs want to leverage spot/preemptible VMs to reduce the cost.
Some pods running legacy Windows applications requires Windows Containers available with Windows VMs.
Some teams want to physically isolate their non-production environments (dev, test, QA, staging, etc.) within the same cluster. This is because it is relatively easier to manage less clusters.
These teams realized that logical isolation with namespaces is not enough. These reasons led to the creation of heterogeneous nodes within the cluster. To make it easier to manage these nodes, Kubernetes introduced the Nodepool. The nodepool is a group of nodes that share the same configuration (CPU, Memory, Networking, OS, maximum number of pods, etc.). By default, one single (system) nodepool is created within the cluster. However, we can add nodepools during or after cluster creation. We can also remove these nodepools at any time. There are 2 types of nodepools:

 

1. System nodepool: used to preferably deploy system pods. Kubernetes could have multiple system nodepools. At least one nodepool is required with at least one single node. System nodepools must run only on Linux due to the dependency to Linux components (no support for Windows).

2. User nodepool: used to preferably deploy application pods. Kubernetes could have multiple user nodepools or none. All user nodepools could scale down to zero nodes. A user nodepool could run on Linux or Windows nodes.

 

Nodepool might have some 'small' different configurations with different cloud providers. This article will focus on Azure Kubernetes Service (AKS). Let's see a demo on how that works with AKS.

 

This article is also available in video format:


1. Create an AKS cluster with one single system nodepool

 

We'll start by creating a new AKS cluster using the Azure CLI:

 

 $ az group create -n aks-cluster -l westeurope                            
 $ az aks create -n aks-cluster -g aks-cluster
 

This will create a new cluster with one single nodepool called agentpool with 3 nodes.

 

 $ kubectl get nodes
 

thumbnail image 1 of blog post titled 
	
	
	 
	
	
	
				
		
			
				
						
							Kubernetes Nodepools Explained
							
						
					
			
		
	
			
	
	
	
	
	

 

This node pool is of type System. It doesn't have any taints. Taints allows a node to ‘accept’ only some pods to be deployed. The only pods that could be deployed are the ones using a specific Toleration. In other words, the node says « I cannot accept any pod except the ones tolerating my taints ». And the pods says « I could be deployed on that node because I have the required toleration ». More details about taints and tolerations here.

 

 $ kubectl get nodes -o json | jq '.items[].spec.taints'
 

thumbnail image 2 of blog post titled 
	
	
	 
	
	
	
				
		
			
				
						
							Kubernetes Nodepools Explained
							
						
					
			
		
	
			
	
	
	
	
	

But it has some labels for its nodes. Let's show the labels with the following command:

 

$ kubectl get nodes -o json | jq '.items[].metadata.labels'
 

thumbnail image 3 of blog post titled 
	
	
	 
	
	
	
				
		
			
				
						
							Kubernetes Nodepools Explained
							
						
					
			
		
	
			
	
	
	
	
	

 

2. Add a new user nodepool for the user applications

 

We'll add a new user nodepool. That could be done using the Azure portal. Go to the cluster, search for Nodepools in the left blade, then click 'add nodepool'.

thumbnail image 4 of blog post titled 
	
	
	 
	
	
	
				
		
			
				
						
							Kubernetes Nodepools Explained
							
						
					
			
		
	
			
	
	
	
	
	

The creation of the nodepool could be done also using the command line which have more options like specifying Spot instances.

 

$ az aks nodepool add `
     --resource-group aks-cluster `
     --cluster-name aks-cluster `
     --name appsnodepool `
     --node-count 5 `
     --node-vm-size Standard_B2ms `
     --kubernetes-version 1.20.7 `
     --max-pods 30 `
     --priority Regular `
     --zones 3 `
     --mode User
 

Note the --priority parameter that could be used with value "Spot" to create Spot VM instances. Spot instances are used for cost optimization.

We can then view the 2 nodepools from the portal or command line.

 

 $ az aks nodepool list --cluster-name aks-cluster --resource-group aks-cluster -o table
 

thumbnail image 5 of blog post titled 
	
	
	 
	
	
	
				
		
			
				
						
							Kubernetes Nodepools Explained
							
						
					
			
		
	
			
	
	
	
	
	

The end result of adding a new nodepool should be like the following:

thumbnail image 6 of blog post titled 
	
	
	 
	
	
	
				
		
			
				
						
							Kubernetes Nodepools Explained
							
						
					
			
		
	
			
	
	
	
	
	

 

3. Deploy an application into a specific nodepool.

 

By default, if we deploy a pod into the cluster, it could be deployed into any of the 2 nodepools.

However, we can choose to target a specific nodepool using Labels on nodepools and nodeSelector from deployment/pods.

Each nodepool have its own set of labels like the agent pool name ("agentpool": "appsnodepool",). We can use the label to target the nodes by using nodeSelector from the deployment file. More details about Labels and nodeSelector here.

Let's show the labels of one of one of the users nodepool nodes with the following command. Make sure to replace the node name.

 

$ kubectl get node aks-appsnodepool-20474252-vmss000001 -o json | jq '.metadata.labels'
 

{
  "agentpool": "appsnodepool",
  "beta.kubernetes.io/arch": "amd64",
  "beta.kubernetes.io/instance-type": "Standard_B2ms",
  "beta.kubernetes.io/os": "linux",
  "failure-domain.beta.kubernetes.io/region": "westeurope",
  "failure-domain.beta.kubernetes.io/zone": "westeurope-3",
  "kubernetes.azure.com/cluster": "MC_aks-cluster_aks-cluster_westeurope",
  "kubernetes.azure.com/node-image-version": "AKSUbuntu-1804gen2containerd-2021.05.19",
  "kubernetes.azure.com/role": "agent",
  "kubernetes.io/arch": "amd64",
  "kubernetes.io/hostname": "aks-appsnodepool-20474252-vmss000001",
  "kubernetes.io/os": "linux",
  "kubernetes.io/role": "agent",
  "node-role.kubernetes.io/agent": "",
  "node.kubernetes.io/instance-type": "Standard_B2ms",
  "storageprofile": "managed",
  "storagetier": "Premium_LRS",
  "topology.kubernetes.io/region": "westeurope",
  "topology.kubernetes.io/zone": "westeurope-3"
}

 

Let's consider the following yaml deployment using the nodeSelector of pool name:

 

# app-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app-deploy
  name: app-deploy
spec:
  replicas: 100
  selector:
    matchLabels:
      app: app-deploy
  template:
    metadata:
      labels:
        app: app-deploy
    spec:
      containers:
      - image: nginx
        name: nginx
      nodeSelector:
        agentpool: appsnodepool
 

Let's deploy the yaml file.

 

$ kubectl apply -f app-deploy.yaml
 

Note how all the 100 pods are all deployed to the same user nodepool.

 

$ kubectl get pods -o wide
 

thumbnail image 7 of blog post titled 
	
	
	 
	
	
	
				
		
			
				
						
							Kubernetes Nodepools Explained
							
						
					
			
		
	
			
	
	
	
	
	

 

4. Deploying system pods into system nodepool

 

The application pods will be deployed only to the user nodepool. However, the system pods could be rescheduled to the user nodepool. We don't want that to happen as we want to physically isolate these critical system pods. The solution here is to use Taints on the nodepool and Tolerations on the pods.

System pods like CoreDNS already have default tolerations like CriticalAddonsOnly.

 

      tolerations:
        - key: CriticalAddonsOnly
          operator: Exists
 

 

$ kubectl get deployment coredns -n kube-system -o json | jq ".spec.template.spec.tolerations"
 

thumbnail image 8 of blog post titled 
	
	
	 
	
	
	
				
		
			
				
						
							Kubernetes Nodepools Explained
							
						
					
			
		
	
			
	
	
	
	
	

To allow these system pods to be deployed only to system nodepool, we need to make sure the system nodepool defines a taint with the same name. As seen earlier, the system nodepool doesn't have any taints by default. Unfortunately, we can add taints only during nodepool creation, not after. So we need to create a new system nodepool with taint (CriticalAddonsOnly=true:NoSchedule).

 

$ az aks nodepool add `
     --resource-group aks-cluster `
     --cluster-name aks-cluster `
     --name systempool `
     --node-count 3 `
     --node-vm-size Standard_D2s_v4 `
     --kubernetes-version 1.20.7 `
     --max-pods 30 `
     --priority Regular `
     --zones 1, 2, 3 `
     --node-taints CriticalAddonsOnly=true:NoSchedule `
     --mode System
 

The same thing could be achieved using the Azure portal :

thumbnail image 9 of blog post titled 
	
	
	 
	
	
	
				
		
			
				
						
							Kubernetes Nodepools Explained
							
						
					
			
		
	
			
	
	
	
	
	

System pods will still run on old system nodepool until we drain that nodepool or delete it.

Let’s go to delete it from the portal or the following command.

 

$ az aks nodepool delete --cluster-name aks-cluster --resource-group aks-cluster --name agentpool
 

Let’s now verify that system pods (except the DaemonSets) are deployed only into the new system nodepool nodes :

 

$ kubectl get pods -n kube-system -o wide
 

thumbnail image 10 of blog post titled 
	
	
	 
	
	
	
				
		
			
				
						
							Kubernetes Nodepools Explained
							
						
					
			
		
	
			
	
	
	
	
	

Note that it is possible to force pods to be scheduled into the system nodepool by adding the following toleration to the pod or the deployment. This should be done to the components like Calico, Prometheus, Elasticsearch…

 

      tolerations:
        - key: CriticalAddonsOnly
          operator: Exists
 

 

5. Using Nodepools to upgrade the cluster with less risk.

 

Upgrading the entire cluster (control plane and all nodepools) might be a risky operation. Nodepools could be leveraged to reduce this risk. Instead of upgrading the nodepool, we proceed with blue/green upgrade:

Upgrade only the control plane to the newer version. 
az aks upgrade –name N --resource-group G --control-plane-only​
Create a new nodepool with the newer k8s version.
Deploy the application pods in the newer nodepool.
Verify the application works fine on the new nodepool.
Delete the old nodepool.
 

Conclusion

Using nodepools we can physically isolate system and user application pods.

It could be also used to isolate different applications, teams, or environments.

It helps also to upgrade the cluster with less risk.

In addition to that it helps to use a new Subnet with different IP range if needed.

And it helps to reset the max pods per node which might be forgotten during cluster creation.

 

Resources:

https://docs.microsoft.com/en-us/azure/aks/use-system-pools

https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools

https://www.youtube.com/watch?v=HJ6F05Pm5mQ&list=PLpbcUe4chE79sB7Jg7B4z3HytqUUEwcNE

 

Disclaimer

The sample scripts are not supported under any Microsoft standard support program or service. The sample scripts are provided AS IS without warranty of any kind. Microsoft further disclaims all implied warranties including, without limitation, any implied warranties of merchantability or of fitness for a particular purpose. The entire risk arising out of the use or performance of the sample scripts and documentation remains with you. In no event shall Microsoft, its authors, or anyone else involved in the creation, production, or delivery of the scripts be liable for any damages whatsoever (including, without limitation, damages for loss of business profits, business interruption, loss of business information, or other pecuniary loss) arising out of the use of or inability to use the sample scripts or documentation, even if Microsoft has been advised of the possibility of such damages.