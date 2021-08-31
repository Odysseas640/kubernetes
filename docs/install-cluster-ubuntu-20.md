# Install Kubernetes Cluster using kubeadm
Follow this documentation to set up a Kubernetes cluster on __Ubuntu 20.04 LTS__.

This documentation guides you in setting up a cluster with one master node and one worker node.

## Assumptions
|Role|IP|OS|RAM|CPU|
|----|----|----|----|----|
|Master|192.168.112.141|Ubuntu 20.04|6G|4|
|Worker 1|192.168.112.140|Ubuntu 20.04|4G|3|
|Worker 2|192.168.112.142|Ubuntu 20.04|4G|3|
|NFS |192.168.112.143|Ubuntu 20.04|3G|2|

## On both Kmaster and Kworker
##### Login as `root` user
```
sudo su -
```
Perform all the commands as root user unless otherwise specified
##### Disable Firewall
```
ufw disable
```
##### Disable swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
##### Update sysctl settings for Kubernetes networking
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
##### Install docker engine
```
{
  apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
  add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  apt update
  apt install -y docker-ce=5:19.03.10~3-0~ubuntu-focal containerd.io
}
```
### Kubernetes Setup
##### Add Apt repository
```
{
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
}
```
##### Install Kubernetes components
```
apt update && apt install -y kubeadm=1.18.5-00 kubelet=1.18.5-00 kubectl=1.18.5-00
```
##### In case you are using LXC containers for Kubernetes nodes
Hack required to provision K8s v1.15+ in LXC containers
```
{
  mknod /dev/kmsg c 1 11
  echo '#!/bin/sh -e' >> /etc/rc.local
  echo 'mknod /dev/kmsg c 1 11' >> /etc/rc.local
  chmod +x /etc/rc.local
}
```

## On kmaster
##### Initialize Kubernetes Cluster
Update the below command with the ip address of kmaster
```
kubeadm init --apiserver-advertise-address=192.168.112.141 --pod-network-cidr=192.168.0.0/16  --ignore-preflight-errors=all
```
##### Deploy Calico network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```

##### Cluster join command
```
kubeadm token create --print-join-command
```

##### To be able to run kubectl commands as non-root user
If you want to be able to run kubectl commands as non-root user, then as a non-root user perform these
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## On Kworker
##### Join the cluster
Use the output from __kubeadm token create__ command in previous step from the master server and run here.

## Verifying the cluster (On kmaster)
##### Get Nodes status
```
kubectl get nodes
```
##### Get component status
```
kubectl get cs
```

(from Stackoverflow) Kubectl commands won't work, do this to fix it:
Check that the api server is actually running and hasn’t crashed:

```
docker ps | grep kube-apiserver
```
But the most likely problem and the one that usually gets me is that you don’t have a .kube directory with the right config in it. Try this:
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Try some nginx
```
kubectl create namespace nginx
kubectl create deploy nginx --image nginx -n nginx
```
```
kubectl expose deploy nginx --port 80 --type NodePort -n nginx
```
```
kubectl get svc
```
```
kubectl get all -o wide
```
```
delete namespace nginx
```
# METALLB
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
```
You might want to edit this metallb.yaml file, to get the controller to run on the master node. I think it makes more sense.
If metallb controller falls into an error loop again, delete namespace metallb-system and install version 9.6.
Run "kubectl get all --all-namespaces" to see if the speakers are working. If they're not, run this:
```
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
Config file so Metallb knows what addresses to hand out.
Save this file somewhere (like the desktop) as metallb.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250
```
Apply configuration
```
kubectl create -f /home/scio/metallb.yaml
```
Install nginx to see if it works
```
kubectl create namespace nginx
kubectl create deploy nginx --image nginx -n nginx
```
```
kubectl expose deploy nginx --port 80 --type LoadBalancer -n nginx
```
```
kubectl delete namespace nginx
```

# NFS SERVER
Make a new virtual machine (NFS Host) and run this
```
sudo apt update
sudo apt install nfs-kernel-server -y
```
```
sudo mkdir /srv/nfs/kubedata -p
sudo chown nobody:nogroup /srv/nfs/kubedata
```
Edit the exports file to broadcast the shared directory
```
sudo nano /etc/exports
```
Copy this in there:
```
/srv/nfs/kubedata \*(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
```
After this there's something about adjusting the firewall, not very important.
https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-20-04

```
sudo systemctl enable --now nfs-server
```
```
sudo exportfs -rav
```
```
sudo showmount -e localhost
```
^ This should show: /srv/nfs/kubedata *

On the NFS client:
```
sudo apt update
sudo apt install nfs-common -y
```
Try to mount the volume to see if it works:
```
mount -t nfs 172.26.7.45:/srv/nfs/kubedata /mnt
```
```
mount | grep kubedata
```
```
umount /mnt
```
Go to https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/v4.0.2
and choose v4.0.2

Install Helm
```
curl https://raw.githubusercontent.com/helm/helm/HEAD/scripts/get-helm-3 | bash
helm version
```
Deploy with Helm:
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
```
```
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=172.26.7.45 --set nfs.path=/srv/nfs/kubedata
helm list
```
```
kubectl get pods
```
```
kubectl get sc
```
Make storage class default:
```
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
Make a PVC:
```
kubectl create -f 4-pvc-nfs.yaml
```
4-pvc-nfs.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-pv1
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
```
```
kubectl get sc,pv,pvc
```
# Jupyterhub YAML:
```
# fullnameOverride and nameOverride distinguishes blank strings, null values,
# and non-blank strings. For more details, see the configuration reference.
fullnameOverride: ""
nameOverride:

# custom can contain anything you want to pass to the hub pod, as all passed
# Helm template values will be made available there.
custom: {}

# imagePullSecret is configuration to create a k8s Secret that Helm chart's pods
# can get credentials from to pull their images.
imagePullSecret:
  create: false
  automaticReferenceInjection: true
  registry:
  username:
  password:
  email:
# imagePullSecrets is configuration to reference the k8s Secret resources the
# Helm chart's pods can get credentials from to pull their images.
imagePullSecrets: []

# hub relates to the hub pod, responsible for running JupyterHub, its configured
# Authenticator class KubeSpawner, and its configured Proxy class
# ConfigurableHTTPProxy. KubeSpawner creates the user pods, and
# ConfigurableHTTPProxy speaks with the actual ConfigurableHTTPProxy server in
# the proxy pod.
hub:
  config:
    JupyterHub:
      admin_access: true
      authenticator_class: dummy
  service:
    type: ClusterIP
    annotations: {}
    ports:
      nodePort:
    extraPorts: []
    loadBalancerIP:
  baseUrl: /
  cookieSecret:
  initContainers: []
  fsGid: 1000
  nodeSelector: {}
  tolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
  concurrentSpawnLimit: 64
  consecutiveFailureLimit: 5
  activeServerLimit:
  deploymentStrategy:
    ## type: Recreate
    ## - sqlite-pvc backed hubs require the Recreate deployment strategy as a
    ##   typical PVC storage can only be bound to one pod at the time.
    ## - JupyterHub isn't designed to support being run in parallell. More work
    ##   needs to be done in JupyterHub itself for a fully highly available (HA)
    ##   deployment of JupyterHub on k8s is to be possible.
    type: Recreate
  db:
    type: sqlite-pvc
    upgrade:
    pvc:
      annotations: {}
      selector: {}
      accessModes:
        - ReadWriteOnce
      storage: 1Gi
      subPath:
      storageClassName:
    url:
    password:
  labels: {}
  annotations: {}
  command: []
  args: []
  extraConfig: {}
  extraFiles: {}
  extraEnv: {}
  extraContainers: []
  extraVolumes: []
  extraVolumeMounts: []
  image:
    name: jupyterhub/k8s-hub
    tag: "1.1.0"
    pullPolicy:
    pullSecrets: []
  resources: {}
  containerSecurityContext:
    runAsUser: 1000
    runAsGroup: 1000
    allowPrivilegeEscalation: false
  lifecycle: {}
  services: {}
  pdb:
    enabled: false
    maxUnavailable:
    minAvailable: 1
  networkPolicy:
    enabled: true
    ingress: []
    ## egress for JupyterHub already includes Kubernetes internal DNS and
    ## access to the proxy, but can be restricted further, but ensure to allow
    ## access to the Kubernetes API server that couldn't be pinned ahead of
    ## time.
    ##
    ## ref: https://stackoverflow.com/a/59016417/2220152
    egress:
      - to:
          - ipBlock:
              cidr: 0.0.0.0/0
    interNamespaceAccessLabels: ignore
    allowedIngressPorts: []
  allowNamedServers: false
  namedServerLimitPerUser:
  authenticatePrometheus:
  redirectToServer:
  shutdownOnLogout:
  templatePaths: []
  templateVars: {}
  livenessProbe:
    # The livenessProbe's aim to give JupyterHub sufficient time to startup but
    # be able to restart if it becomes unresponsive for ~5 min.
    enabled: true
    initialDelaySeconds: 300
    periodSeconds: 10
    failureThreshold: 30
    timeoutSeconds: 3
  readinessProbe:
    # The readinessProbe's aim is to provide a successful startup indication,
    # but following that never become unready before its livenessProbe fail and
    # restarts it if needed. To become unready following startup serves no
    # purpose as there are no other pod to fallback to in our non-HA deployment.
    enabled: true
    initialDelaySeconds: 0
    periodSeconds: 2
    failureThreshold: 1000
    timeoutSeconds: 1
  existingSecret:
  serviceAccount:
    annotations: {}
  extraPodSpec: {}

rbac:
  enabled: true

# proxy relates to the proxy pod, the proxy-public service, and the autohttps
# pod and proxy-http service.
proxy:
  secretToken: "6888ff37aeb1f18df54983b7f8fa00467a30dcdffd28c661a96dbefa01534a7b"
  annotations: {}
  deploymentStrategy:
    ## type: Recreate
    ## - JupyterHub's interaction with the CHP proxy becomes a lot more robust
    ##   with this configuration. To understand this, consider that JupyterHub
    ##   during startup will interact a lot with the k8s service to reach a
    ##   ready proxy pod. If the hub pod during a helm upgrade is restarting
    ##   directly while the proxy pod is making a rolling upgrade, the hub pod
    ##   could end up running a sequence of interactions with the old proxy pod
    ##   and finishing up the sequence of interactions with the new proxy pod.
    ##   As CHP proxy pods carry individual state this is very error prone. One
    ##   outcome when not using Recreate as a strategy has been that user pods
    ##   have been deleted by the hub pod because it considered them unreachable
    ##   as it only configured the old proxy pod but not the new before trying
    ##   to reach them.
    type: Recreate
    ## rollingUpdate:
    ## - WARNING:
    ##   This is required to be set explicitly blank! Without it being
    ##   explicitly blank, k8s will let eventual old values under rollingUpdate
    ##   remain and then the Deployment becomes invalid and a helm upgrade would
    ##   fail with an error like this:
    ##
    ##     UPGRADE FAILED
    ##     Error: Deployment.apps "proxy" is invalid: spec.strategy.rollingUpdate: Forbidden: may not be specified when strategy `type` is 'Recreate'
    ##     Error: UPGRADE FAILED: Deployment.apps "proxy" is invalid: spec.strategy.rollingUpdate: Forbidden: may not be specified when strategy `type` is 'Recreate'
    rollingUpdate:
  # service relates to the proxy-public service
  service:
    type: LoadBalancer
    labels: {}
    annotations: {}
    nodePorts:
      http:
      https:
    disableHttpPort: false
    extraPorts: []
    loadBalancerIP:
    loadBalancerSourceRanges: []
  # chp relates to the proxy pod, which is responsible for routing traffic based
  # on dynamic configuration sent from JupyterHub to CHP's REST API.
  chp:
    containerSecurityContext:
      runAsUser: 65534 # nobody user
      runAsGroup: 65534 # nobody group
      allowPrivilegeEscalation: false
    image:
      name: jupyterhub/configurable-http-proxy
      tag: 4.5.0 # https://github.com/jupyterhub/configurable-http-proxy/releases
      pullPolicy:
      pullSecrets: []
    extraCommandLineFlags: []
    livenessProbe:
      enabled: true
      initialDelaySeconds: 60
      periodSeconds: 10
    readinessProbe:
      enabled: true
      initialDelaySeconds: 0
      periodSeconds: 2
      failureThreshold: 1000
    resources: {}
    defaultTarget:
    errorTarget:
    extraEnv: {}
    nodeSelector: {}
    tolerations:
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"
      effect: "NoSchedule"
    networkPolicy:
      enabled: true
      ingress: []
      egress:
        - to:
            - ipBlock:
                cidr: 0.0.0.0/0
      interNamespaceAccessLabels: ignore
      allowedIngressPorts: [http, https]
    pdb:
      enabled: false
      maxUnavailable:
      minAvailable: 1
    extraPodSpec: {}
  # traefik relates to the autohttps pod, which is responsible for TLS
  # termination when proxy.https.type=letsencrypt.
  traefik:
    containerSecurityContext:
      runAsUser: 65534 # nobody user
      runAsGroup: 65534 # nobody group
      allowPrivilegeEscalation: false
    image:
      name: traefik
      tag: v2.4.11 # ref: https://hub.docker.com/_/traefik?tab=tags
      pullPolicy:
      pullSecrets: []
    hsts:
      includeSubdomains: false
      preload: false
      maxAge: 15724800 # About 6 months
    resources: {}
    labels: {}
    extraEnv: {}
    extraVolumes: []
    extraVolumeMounts: []
    extraStaticConfig: {}
    extraDynamicConfig: {}
    nodeSelector: {}
    tolerations:
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"
      effect: "NoSchedule"
    extraPorts: []
    networkPolicy:
      enabled: true
      ingress: []
      egress:
        - to:
            - ipBlock:
                cidr: 0.0.0.0/0
      interNamespaceAccessLabels: ignore
      allowedIngressPorts: [http, https]
    pdb:
      enabled: false
      maxUnavailable:
      minAvailable: 1
    serviceAccount:
      annotations: {}
    extraPodSpec: {}
  secretSync:
    containerSecurityContext:
      runAsUser: 65534 # nobody user
      runAsGroup: 65534 # nobody group
      allowPrivilegeEscalation: false
    image:
      name: jupyterhub/k8s-secret-sync
      tag: "1.1.0"
      pullPolicy:
      pullSecrets: []
    resources: {}
  labels: {}
  https:
    enabled: false
    type: letsencrypt
    #type: letsencrypt, manual, offload, secret
    letsencrypt:
      contactEmail:
      # Specify custom server here (https://acme-staging-v02.api.letsencrypt.org/directory) to hit staging LE
      acmeServer: https://acme-v02.api.letsencrypt.org/directory
    manual:
      key:
      cert:
    secret:
      name:
      key: tls.key
      crt: tls.crt
    hosts: []

# singleuser relates to the configuration of KubeSpawner which runs in the hub
# pod, and its spawning of user pods such as jupyter-myusername.
singleuser:
  podNameTemplate:
  extraTolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
  nodeSelector:
    kubernetes.io/hostname: k8worker
  extraNodeAffinity:
    required: []
    preferred: []
  extraPodAffinity:
    required: []
    preferred: []
  extraPodAntiAffinity:
    required: []
    preferred: []
  networkTools:
    image:
      name: jupyterhub/k8s-network-tools
      tag: "1.1.0"
      pullPolicy:
      pullSecrets: []
  cloudMetadata:
    # block set to true will append a privileged initContainer using the
    # iptables to block the sensitive metadata server at the provided ip.
    blockWithIptables: true
    ip: 169.254.169.254
  networkPolicy:
    enabled: true
    ingress: []
    egress:
      # Required egress to communicate with the hub and DNS servers will be
      # augmented to these egress rules.
      #
      # This default rule explicitly allows all outbound traffic from singleuser
      # pods, except to a typical IP used to return metadata that can be used by
      # someone with malicious intent.
      - to:
          - ipBlock:
              cidr: 0.0.0.0/0
              except:
                - 169.254.169.254/32
    interNamespaceAccessLabels: ignore
    allowedIngressPorts: []
  events: true
  extraAnnotations: {}
  extraLabels:
    hub.jupyter.org/network-access-hub: "true"
  extraFiles: {}
  extraEnv: {}
  lifecycleHooks: {}
  initContainers: []
  extraContainers: []
  uid: 1000
  fsGid: 100
  serviceAccountName:
  storage:
    type: dynamic
    extraLabels: {}
    extraVolumes: []
    extraVolumeMounts: []
    static:
      pvcName:
      subPath: "{username}"
    capacity: 10Gi
    homeMountPath: /home/jovyan
    dynamic:
      storageClass:
      pvcNameTemplate: claim-{username}{servername}
      volumeNameTemplate: volume-{username}{servername}
      storageAccessModes: [ReadWriteOnce]
  image:
    name: jupyterhub/k8s-singleuser-sample
    tag: "1.1.0"
    pullPolicy:
    pullSecrets: []
  startTimeout: 300
  cpu:
    limit:
    guarantee:
  memory:
    limit:
    guarantee: 1G
  extraResource:
    limits: {}
    guarantees: {}
  cmd: jupyterhub-singleuser
  defaultUrl:
  extraPodConfig: {}
  profileList: []

# scheduling relates to the user-scheduler pods and user-placeholder pods.
scheduling:
  userScheduler:
    enabled: true
    replicas: 2
    logLevel: 4
    # plugins ref: https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins-1
    plugins:
      score:
        disabled:
          - name: SelectorSpread
          - name: TaintToleration
          - name: PodTopologySpread
          - name: NodeResourcesBalancedAllocation
          - name: NodeResourcesLeastAllocated
          # Disable plugins to be allowed to enable them again with a different
          # weight and avoid an error.
          - name: NodePreferAvoidPods
          - name: NodeAffinity
          - name: InterPodAffinity
          - name: ImageLocality
        enabled:
          - name: NodePreferAvoidPods
            weight: 161051
          - name: NodeAffinity
            weight: 14631
          - name: InterPodAffinity
            weight: 1331
          - name: NodeResourcesMostAllocated
            weight: 121
          - name: ImageLocality
            weight: 11
    containerSecurityContext:
      runAsUser: 65534 # nobody user
      runAsGroup: 65534 # nobody group
      allowPrivilegeEscalation: false
    image:
      # IMPORTANT: Bumping the minor version of this binary should go hand in
      #            hand with an inspection of the user-scheduelrs RBAC resources
      #            that we have forked.
      name: k8s.gcr.io/kube-scheduler
      tag: v1.19.13 # ref: https://github.com/kubernetes/website/blob/main/content/en/releases/patch-releases.md
      pullPolicy:
      pullSecrets: []
    nodeSelector: {}
    tolerations:
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"
      effect: "NoSchedule"
    pdb:
      enabled: true
      maxUnavailable: 1
      minAvailable:
    resources: {}
    serviceAccount:
      annotations: {}
    extraPodSpec: {}
  podPriority:
    enabled: false
    globalDefault: false
    defaultPriority: 0
    userPlaceholderPriority: -10
  userPlaceholder:
    enabled: true
    image:
      name: k8s.gcr.io/pause
      # tag's can be updated by inspecting the output of the command:
      # gcloud container images list-tags k8s.gcr.io/pause --sort-by=~tags
      #
      # If you update this, also update prePuller.pause.image.tag
      tag: "3.5"
      pullPolicy:
      pullSecrets: []
    replicas: 0
    containerSecurityContext:
      runAsUser: 65534 # nobody user
      runAsGroup: 65534 # nobody group
      allowPrivilegeEscalation: false
    resources: {}
  corePods:
    tolerations:
      - key: hub.jupyter.org/dedicated
        operator: Equal
        value: core
        effect: NoSchedule
      - key: hub.jupyter.org_dedicated
        operator: Equal
        value: core
        effect: NoSchedule
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
    nodeAffinity:
      matchNodePurpose: prefer
  userPods:
    tolerations:
      - key: hub.jupyter.org/dedicated
        operator: Equal
        value: user
        effect: NoSchedule
      - key: hub.jupyter.org_dedicated
        operator: Equal
        value: user
        effect: NoSchedule
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
    nodeAffinity:
      matchNodePurpose: prefer

# prePuller relates to the hook|continuous-image-puller DaemonsSets
prePuller:
  annotations: {}
  resources: {}
  containerSecurityContext:
    runAsUser: 65534 # nobody user
    runAsGroup: 65534 # nobody group
    allowPrivilegeEscalation: false
  extraTolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
  # hook relates to the hook-image-awaiter Job and hook-image-puller DaemonSet
  hook:
    enabled: true
    pullOnlyOnChanges: true
    # image and the configuration below relates to the hook-image-awaiter Job
    image:
      name: jupyterhub/k8s-image-awaiter
      tag: "1.1.0"
      pullPolicy:
      pullSecrets: []
    containerSecurityContext:
      runAsUser: 65534 # nobody user
      runAsGroup: 65534 # nobody group
      allowPrivilegeEscalation: false
    podSchedulingWaitDuration: 10
    nodeSelector: {}
    tolerations:
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"
      effect: "NoSchedule"
    resources: {}
    serviceAccount:
      annotations: {}
  continuous:
    enabled: true
  pullProfileListImages: true
  extraImages: {}
  pause:
    containerSecurityContext:
      runAsUser: 65534 # nobody user
      runAsGroup: 65534 # nobody group
      allowPrivilegeEscalation: false
    image:
      name: k8s.gcr.io/pause
      # tag's can be updated by inspecting the output of the command:
      # gcloud container images list-tags k8s.gcr.io/pause --sort-by=~tags
      #
      # If you update this, also update scheduling.userPlaceholder.image.tag
      tag: "3.5"
      pullPolicy:
      pullSecrets: []

ingress:
  enabled: false
  annotations: {}
  hosts: []
  pathSuffix:
  pathType: Prefix
  tls: []

# cull relates to the jupyterhub-idle-culler service, responsible for evicting
# inactive singleuser pods.
#
# The configuration below, except for enabled, corresponds to command-line flags
# for jupyterhub-idle-culler as documented here:
# https://github.com/jupyterhub/jupyterhub-idle-culler#as-a-standalone-script
#
cull:
  enabled: true
  users: false # --cull-users
  removeNamedServers: false # --remove-named-servers
  timeout: 3600 # --timeout
  every: 600 # --cull-every
  concurrency: 10 # --concurrency
  maxAge: 0 # --max-age

debug:
  enabled: false

global:
  safeToShowValues: false
```
```
helm upgrade --cleanup-on-fail --install jupyterhub jupyterhub/jupyterhub --namespace jhub --create-namespace --values /home/odysseas/Desktop/jupyterhub.yaml
```

Useful stuff:
```
kubectl get all -o wide --all-namespaces
```
```
kubectl delete namespace jhub
```
```
kubectl get pods -o wide --all-namespaces
```
## Assign labels to nodes:
```
kubectl label nodes k8worker put_jupyter_notebook_here=ye
```
Go to NodeSelector in the YAML and set it. It should look like this:
```
singleuser:
  nodeSelector:
    jupyter_notebook_here: ye
```
If the worker VMs have 4 GB of RAM, set the notebook to require 3 so it'll have to go to a different worker. Otherwise it'll go in the same one.
## Get pods to tolerate master:
```
singleuser:
  extraTolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
  nodeSelector:
```
# Install the company's single user notebook:
```
docker pull scioquiver/notebooks:cgspatial-notebook
```
Run this command
```
docker images
```
And copy the output (where it says scioquiver) into the big ass YAML file. It should look like this:
```
singleuser:
.........
  image:
    name: scioquiver/notebooks
    tag: cgspatial-notebook
.........
```
Run this (exactly as before):
```
helm upgrade --cleanup-on-fail --install jupyterhub jupyterhub/jupyterhub --namespace jhub --create-namespace --values /home/odysseas/Desktop/jupyterhub.yaml
```
and wait for, like, half an hour (the first time, from the second time and on it should be very quick).

I have a 50 Mbit connection and it was maxed out all throughout. I saw free space on the hard drive go down by about 20 GB.

If it fails and says it timed out waiting, run it again.
