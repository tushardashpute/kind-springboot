# kind_springboot

**Step 1: Create Kind Cluster**

Install and start Docker daemon

        yum install docker -y && service docker start

Install kubectl 

        curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.17/2023-11-14/bin/linux/amd64/kubectl
        chmod +x kubectl
        mv kubectl /usr/local/bin/kubectl

Install Kind on Linux

        curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.14.0/kind-linux-amd64
        chmod +x ./kind
        sudo mv ./kind /bin/kind
        
Create a kind cluster with extraPortMappings and node-labels.

extraPortMappings allow the local host to make requests to the Ingress controller over ports 80/443
node-labels only allow the ingress controller to run on a specific node(s) matching the label selector

        cat <<EOF | kind create cluster --config=-
        kind: Cluster
        apiVersion: kind.x-k8s.io/v1alpha4
        nodes:
        - role: control-plane
          kubeadmConfigPatches:
          - |
            kind: InitConfiguration
            nodeRegistration:
              kubeletExtraArgs:
                node-labels: "ingress-ready=true"
          extraPortMappings:
          - containerPort: 80
            hostPort: 80
            protocol: TCP
          - containerPort: 443
            hostPort: 443
            protocol: TCP
        - role: worker
        - role: worker
        EOF

The output shows the progress of the operation. When the cluster successfully initiates, the command prompt appears.

![image](https://github.com/tushardashpute/open_ssl_nginx_kind_cluster/assets/74225291/b00fb1fe-10c5-4d27-bafe-f210edf67d34)

        kubectl get nodes
        NAME                 STATUS   ROLES           AGE   VERSION
        kind-control-plane   Ready    control-plane   47s   v1.24.0
        kind-worker          Ready    <none>          26s   v1.24.0
        kind-worker2         Ready    <none>          26s   v1.24.0


**Step 2: Deploy nginx Ingress controller**

        $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
        
        $ kubectl wait --namespace ingress-nginx \
          --for=condition=ready pod \
          --selector=app.kubernetes.io/component=controller \
          --timeout=90s
        
        ...
        
        pod/ingress-nginx-controller-6f9b5dd966-zkj2c condition met

**Step 3 : Deploy and test sample application**

# apply ingress test manifests

        kubectl apply -f https://raw.githubusercontent.com/tushardashpute/kind-springboot/main/springboot-deployment.yaml
        kubectl apply -f https://raw.githubusercontent.com/tushardashpute/kind-springboot/main/springboot-service.yaml
        kubectl apply -f https://raw.githubusercontent.com/tushardashpute/kind-springboot/main/springboot-ingress.yaml

# Check the pod,service,ingress

        kubectl get ing,svc,pod
        NAME                                           CLASS    HOSTS           ADDRESS     PORTS   AGE
        ingress.networking.k8s.io/springboot-ingress   <none>   ingress.local   localhost   80      12m
        
        NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
        service/kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP     21m
        service/springboot   ClusterIP   10.96.10.44   <none>        33333/TCP   8s
        
        NAME                              READY   STATUS    RESTARTS   AGE
        pod/springboot-746db777fb-wlb2k   1/1     Running   0          18m

Now we can access application by either locally or by using node public IP:

        [root@ip-172-31-86-22 ~]# curl http://localhost/listallcustomers
        [{"name":"vamsi","id":"001","country_of_birth":"INDIA","country_of_residence":"AP","segment":"retail"}]

To access the app from browser open port 80:

![image](https://github.com/tushardashpute/kind-springboot/assets/74225291/a9e7a420-4489-4305-a918-b206da828f59)
