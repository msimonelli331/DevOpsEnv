# DevOpsEnv

Helm charts and steps for a Kuberenetes DevOps Environment

## Dependencies

To use these helm charts you need a kubernetes cluster. For this example we're going to use Rocky Linux and microk8s

### Rocky Linux

#### Preconditions

1. Internet Connection
2. Hypervisor

#### Install Steps

1. Download Rocky Linux v9.4 Boot ISO from [here](https://rockylinux.org/download)
2. Attach the ISO to a new VM or make a bootable USB

   - Suggested VM specs:

     1. 20480 MB of RAM
     2. 4 CPUs
     3. 100 GB of Storage

3. Choose minimal install
4. Create an admin user
5. Once booted, setup the root user password

   ```bash
   sudo su -
   passwd
   ```

6. Disable firewalld, causes routing problems between pods

   ```bash
   systemctl disable --now firewalld
   ```

7. Install snapd by following steps from [here](https://snapcraft.io/docs/installing-snap-on-rocky)
   ```bash
   dnf install epel-release -y
   dnf install snapd -y
   systemctl start snapd.socket
   ln -s /var/lib/snapd/snap /snap
   ```

### MicroK8s

#### Preconditions

1. Internet Connection
2. OS with snapd installed

#### Install Steps

1. Install microk8s with snap, following steps from [here](https://microk8s.io/docs/getting-started)

   ```bash
   snap install microk8s --classic --channel=1.31
   ```

2. Setup permissions to use microk8s command

   ```bash
   usermod -a -G microk8s $USER
   mkdir -p ~/.kube
   chmod 0700 ~/.kube
   su - $USER
   ```

3. Setup alias' to use commands normally

   ```bash
   snap alias microk8s.kubectl kubectl
   snap alias microk8s.helm helm
   ```

4. Add an ingress controller by following steps from [here](https://kubernetes.github.io/ingress-nginx/deploy/#microk8s) and [here](https://microk8s.io/docs/addon-ingress)

   ```bash
   microk8s enable ingress
   ```

### Helm

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm

#### Install Steps

1. Add this helm repo

   ```bash
   helm repo add devopsenv https://msimonelli331.github.io/DevOpsEnv/
   ```

2. Confirm the repo was added and has charts

   ```bash
   helm search repo devopsenv
   ```

### CoreDNS

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added

#### Install Steps

1. Disable microk8s coredns

   ```bash
   microk8s disable dns
   ```

2. Install coredns, be sure to change the IP to your machines IP

   ```bash
   helm install coredns devopsenv/coredns --namespace kube-system --set vars.domain=devops --set vars.ip=127.0.0.1
   ```

3. Setup TLS for ingresses following steps [here](https://arminreiter.com/2022/01/create-your-own-certificate-authority-ca-using-openssl/)

   ```bash
   openssl genrsa -aes256 -out ca.key 4096

   openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/C=<Country 2 letter code>/ST=<State 2 letter code>/L=<City>/O=DevOpsEnv/CN=DevOpsEnv Root CA"

   openssl req -new -nodes -out server.csr -newkey rsa:4096 -keyout server.key -subj '/CN=DevOpsEnv Server/C=<Country 2 letter code>/ST=<State 2 letter code>/L=<City>/O=DevOpsEnv'

   openssl req -x509 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256 -subj "/CN=devops/O=DevOpsEnv" -addext "subjectAltName = DNS.1:devops,DNS.2:*.devops"

   openssl x509 -in server.crt -text -noout

   kubectl create secret tls ingress-tls -n devops --cert=server.crt --key=server.key
   ```

### Nexus

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added
4. CoreDNS updated to resolve ingress hostnames
   - A separate hostname for the cotnainer repo is required to docker login
   - You cannot login to nexus.devops/repository/container
     - docker strips the /'s and just tries to login to nexus.devops
   - So the container repo needs its own hostname
     - It is likely possible to use the service name and port in a pod or the endpoint ip and port on the host, but it is convienent to use the ingress name in and outside pods
5. server cert generated that has a SAN for the ingress domain
   - https is required to docker login to the container repo

#### Install Steps

1. Create the host volume directory

   ```bash
   mkdir -p /mnt/devops/nexus
   ```

2. Label the node this local volume is running on

   ```bash
   kubectl label nodes localhost.localdomain nexus=local
   ```

3. Install the helm chart, updating the hostnames to match the selected domain name from previous steps

   ```bash
   helm install nexus devopsenv/nexus --create-namespace -n devops --set nexusHostname="&nexusHostname nexus.devops" --set containerHostname="&containerHostname container.devops"
   ```

4. Get the default password

   ```bash
   kubectl exec -it -n devops deploy/nexus -- cat /nexus-data/admin.password
   ```

5. Login to nexus with the password from the last step, change the password

6. Create a docker hosted container repo
   1. Enable HTTP and use the port 8082

### Gitea

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added

#### Install Steps

1. Create the host volume directory

   ```bash
   mkdir -p /mnt/devops/gitea/{data,config}
   ```

2. Label the node this local volume is running on

   ```bash
   kubectl label nodes localhost.localdomain gitea=local
   ```

3. Update volume permissions

   ```bash
   chown -R 1000:1000 /mnt/devops/gitea
   ```

4. Install the helm chart, updating the hostname to match the selected domain name from previous steps

   ```bash
   helm install gitea devopsenv/gitea --create-namespace -n devops --set gitHostname="&gitHostname git.devops"
   ```

5. Navigate to http://git.devops/ for initial configuration
   1. Configure the database as sqlite3
   2. Add an admin user

### Gitea Runner

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added
4. Gitea deployed and running

#### Install Steps

1. Create the host volume directory

   ```bash
   mkdir -p /mnt/devops/gitea-runner
   ```

2. Label the node this local volume is running on

   ```bash
   kubectl label nodes localhost.localdomain gitea-runner=local
   ```

3. Update volume permissions

   ```bash
   chown -R 1000:1000 /mnt/devops/gitea-runner
   ```

4. Go to gitea webpage > Settings > Actions > Runners > Create new Runner > Copy the token

5. Install the helm chart, updating the hostname to match the selected domain name from previous steps

   ```bash
   helm install gitea-runner devopsenv/gitea-runner --create-namespace -n devops --set token=<registration token from previous step> --set gitURL="&gitURL http://git.devops"
   ```

### Buildkitd

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. This helm repo added

#### Install Steps

4. Install the helm chart

   ```bash
   helm install buildkitd devopsenv/buildkitd --create-namespace -n devops
   ```

### Builder

#### Preconditions

1. Internet Connection
2. OS with Kubernetes and helm
3. Gitea deployed and running
4. Gitea runner deployed, running, and connected to gitea

#### Install Steps

1. Create a Buidler git repo in gitea

2. Add the contents of the builder folder

3. Update the ca.crt with the one generated earlier

## Resources

- https://rockylinux.org/download
- https://snapcraft.io/docs/installing-snap-on-rocky
- https://microk8s.io/docs/getting-started
- https://kubernetes.github.io/ingress-nginx/deploy/#microk8s
- https://microk8s.io/docs/addon-ingress
- https://arminreiter.com/2022/01/create-your-own-certificate-authority-ca-using-openssl/
- https://docs.gitea.com/installation/install-with-docker#configure-the-user-inside-gitea-using-environment-variables
- https://discuss.kubernetes.io/t/microk8s-images-prune-utility-for-production-servers/15874
- https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md
- https://gitea.com/gitea/act_runner/src/branch/main/examples/kubernetes/rootless-docker.yaml

## Notes

Cleanup old images

```bash
VERSION="v1.30.0"
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-amd64.tar.gz --output crictl-${VERSION}-linux-amd64.tar.gz
tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
crictl -r unix:///var/snap/microk8s/common/run/containerd.sock rmi --prune
```
