PRE-REQUISITOS:

1. Crear VPC

    Desde la consola de AWS utilizar la herramienta de creacion de redes para obtener una VPC con esta configuracion, o una similar:

    3 Subnets Publicas
    6 Subnets Privadas
    1 NAT Gateway
    Routables publicas y privadas


2. Instalar kubectl

https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

    curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.25.6/2023-01-30/bin/darwin/amd64/kubectl

    chmod +x ./kubectl

    mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH

    (OPTIONAL) echo 'export PATH=$PATH:$HOME/bin' >> ~/.bash_profile

    kubectl version --short --client # Client Version: v1.25.6


2. Instalar eksctl

https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html

    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"

    brew upgrade eksctl && { brew link --overwrite eksctl; } || { brew tap weaveworks/tap; brew install weaveworks/tap/eksctl; }

    eksctl version # 0.154.0


3. Crear Usuario IAM

 Desde la consola de AWS crear un usuario IAM y asignar mínimo estas políticas: https://eksctl.io/usage/minimum-iam-policies

      export AWS_ACCESS_KEY_ID=AKIAR4R3DH3LLXXXXX
      
      export AWS_SECRET_ACCESS_KEY=h/aKJ3JNdM28kGnHlSXv6/8v9cEuXXXXXXXXX

4. Consultar AMI optimizada para EKS

https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html

    AWS tiene AMI's optimizadas para ejecutar EKS segun la version que se quiera ejecutar, para efectos de este tutorial se estará usando la versión 1.23 de Kubernetes:

    https://us-east-1.console.aws.amazon.com/systems-manager/parameters/aws/service/eks/optimized-ami/1.23/amazon-linux-2-gpu/recommended/image_id/description?region=us-east-1#

    ami-02759925ad596a5f9

5. Crear una Key Pair

    Desde la consola de AWS crear una key pair.

    demo-eks-cluster

    chmod 400 demo-eks-cluster.pem


CREAR EL CLUSTER:

Luego de cumplir con estos pre-requisitos, es el momento de desplegar el cluster de EKS. Para esto, se requiere configurar un archivo `cluster.yaml`, por ejemplo:

    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig
    metadata:
      name: agosto-eks-cluster
      region: us-east-1
      version: "1.25"
    vpc:
      id: "vpc-08f9cdf8d36e48a48"
      subnets:
        public:
          pub-us-east-1a:
            id: "subnet-09a074779c18ff743"
          pub-us-east-1b:
            id: "subnet-0fb31fb16af610114"
          pub-us-east-1c:
            id: "subnet-0a99b1b0714269d6a"
        private:
          priv0-us-east-1a: 
            id: "subnet-0b308209c5695bf3b"
          priv0-us-east-1b: 
            id: "subnet-0dc29ca8e8134daa7"
          priv0-us-east-1c: 
            id: "subnet-03015c0ae9adb98f8"
          priv1-us-east-1a: 
            id: "subnet-0656e14a67ffe653d"
          priv1-us-east-1b: 
            id: "subnet-0e99a3a5a9299615a"
          priv1-us-east-1c: 
            id: "subnet-0845dca73f191f6be"
    managedNodeGroups:
      - name: dev-ng-1
        ami: ami-02759925ad596a5f9
        amiFamily: AmazonLinux2
        overrideBootstrapCommand: |
          #!/bin/bash
          /etc/eks/bootstrap.sh agosto-eks-cluster
        instanceType: t3a.medium
        spot: true
        minSize: 1
        maxSize: 5
        desiredCapacity: 3
        volumeSize: 20
        volumeEncrypted: true
        volumeType: gp3
        privateNetworking: true
        subnets:
          - priv0-us-east-1a
          - priv0-us-east-1b
          - priv0-us-east-1c
          - priv1-us-east-1a
          - priv1-us-east-1b
          - priv1-us-east-1c
        labels: {role: dev}
        ssh:
          publicKeyName: agosto-demo
        tags:
          bootcamp: kubernetes
        iam:
          attachPolicyARNs:
            - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
            - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
            - arn:aws:iam::aws:policy/ElasticLoadBalancingFullAccess
            - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
            - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
          withAddonPolicies:
            autoScaler: true
    iam:
      withOIDC: true
    
    addons:
    - name: vpc-cni
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
      resolveConflicts: overwrite
    - name: coredns
      version: v1.9.3-eksbuild.5
      configurationValues: |-
        replicaCount: 2
      resolveConflicts: overwrite
    - name: kube-proxy
      version: v1.25.11-minimal-eksbuild.2
      resolveConflicts: overwrite


Ahora ejecutar el siguiente comando para testear la configuración:

    eksctl create cluster -f cluster.yaml --dry-run

y finalmente

    eksctl create cluster -f cluster.yaml

Para poder implementar el addon de EBS es necesario ejecutar estos comandos:

    eksctl create iamserviceaccount \
                      --name ebs-csi-controller-sa \
                      --namespace kube-system \
                      --cluster demo-eks-cluster \
                      --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
                      --approve \
                      --role-only \
                      --role-name AmazonEKS_EBS_CSI_DriverRole

    eksctl create addon --name aws-ebs-csi-driver --cluster demo-eks-cluster --service-account-role-arn arn:aws:iam::130046705366:role/AmazonEKS_EBS_CSI_DriverRole --force

Con estos pasos un cluster de EKS debería estar ejecutandose sin problema en la cuenta de AWS asociada.


CONFIGURAR EL ACCESO AL CLUSTER DE EKS:

Para poder acceder a nuestro cluster requerimos de un archivo de configuracion kubeconfig:

    aws eks update-kubeconfig --name demo-eks-cluster --region us-east-1

    kubectl config get-contexts

    kubectl config rename-context arn:aws:eks:us-east-1:130046705366:cluster/demo-eks-cluster demo-eks-cluster

    kubectl config use-context demo-eks-cluster


Descargar e instalar Lens:

    https://k8slens.dev/

    https://k9scli.io/
