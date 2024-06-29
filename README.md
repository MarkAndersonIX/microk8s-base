## Base microk8s Project for Raspberry Pi Cluster ##

## Setup ##
Setting up a microservices architecture on a cluster of Raspberry Pis using Ubuntu Server, MicroK8s, Docker, NGINX, Prometheus, and Jenkins.

1. Install Ubuntu Server on Raspberry Pis
3. Install Docker
    sudo apt-get install -y docker.io
    sudo systemctl enable --now docker
4. Install MicroK8s and Dependencies
    Install:
        sudo snap install microk8s --classic
    Add Your User to the MicroK8s Group:
        sudo usermod -aG microk8s $USER
        sudo chown -f -R $USER ~/.kube
    Enable Required MicroK8s Add-ons:
        microk8s enable dns dashboard storage ingress
5. Configure MicroK8s Cluster
    Initialize the First Node (Master):
    This command will output a join command that you will use on the worker nodes.
        microk8s add-node
    Join Worker Nodes:
    On each worker node, run the join command provided by the master node. Example:
        microk8s join <master-node-ip>:<port>/<token>
6. Deploy Your Microservices
    Create a dockerfile for the microservice, build, and push to a container registry.
        https://hub.docker.com/repository/docker/markandersonix/hello_db/general
    Create Kubernetes Deployments and Services for each microservice:
        [hellodb-deployment](hello_db/hellodb-deployment.yaml)
        [hellodb-service](hello_db/hellodb-service.yaml)
    Apply changes to deploy:
        kubectl apply -f hellodb-deployment.yaml
        kubectl apply -f hellodb-service.yaml
    Deployment management:
        kubectl rollout pause deployment hello-db
        kubectl rollout resume deployment hello-db
        kubectl rollout restart deployment hello-db
7. Kubectl Dashboard:
    This will provide a dashboard surface with information about the cluster.
    Installed via
        microk8s enable dashboard
        microk8s dashboard-proxy
    1. Navigate to the URL, which VSCode should portforward to localhost for you.
    2. Copy the provided token to auth
    
<!-- TODO -->
8.  Deploy NGINX as a Reverse Proxy
    Create a ConfigMap for NGINX configuration:
        [nginx-config](nginx-config.yaml)
    Apply the ConfigMap to the Kubernetes cluster:
        kubectl apply -f nginx-config.yaml
    Create a Deployment for NGINX:
        [nginix-deployment](nginx/nginx-deployment.yaml)
    Apply the Deployment to the Kubernetes cluster:
        kubectl apply -f nginx-deployment.yaml
9.  Set Up Prometheus for Monitoring
    sudo snap install helm --classic
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    helm install prometheus prometheus-community/prometheus
10. Set Up Jenkins for CI/CD
    Deploy Jenkins using Helm:
        helm repo add jenkins https://charts.jenkins.io
        helm repo update
        helm install jenkins jenkins/jenkins
    Access Jenkins:

    Get the admin password:
        kubectl get secret --namespace default jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode
    Access Jenkins at http://<Node-IP>:<NodePort> (get the NodePort by running kubectl get svc).

    Set Up Jenkins:

    Follow the setup wizard to unlock Jenkins, install recommended plugins, and create your first admin user.
    Create a Jenkins Pipeline:

    Create a new pipeline job in Jenkins.

    Use a Jenkinsfile to define your build, test, and deploy steps:

        pipeline {
            agent any
            stages {
                stage('Build') {
                    steps {
                        script {
                            docker.build('catalog', './catalog')
                            docker.build('orders', './orders')
                            docker.build('users', './users')
                        }
                    }
                }
                stage('Deploy') {
                    steps {
                        script {
                            sh 'kubectl rollout restart deployment/catalog-deployment'
                            sh 'kubectl rollout restart deployment/orders-deployment'
                            sh 'kubectl rollout restart deployment/users-deployment'
                        }
                    }
                }
            }
        }
11. Permissions management.