# SmartM2M Technical Test

## Ansible
1. Create an inventory file in Ansible, group your inventory to follow these criteria:
node1 should be the member of "loadbalancer" group.
node2 and node3 should be the member of "backend" group.
2. After creating the inventory, create a playbook to install nginx on all of the servers.
Then, create another playbook to change the default nginx index page with a html file
contains "served from: <hostname>". perform this task on the backend group
expected result: when we hit http://node2, we should get "served from: node2"
3. Lastly, create playbook to setup nginx in the loadbalancer group as follows:
- Make it load balance between nodes in the backend group
- Use "least connection" load balancing method

### Step 1: Inventory Setup

First, we make inventory file for Ansible. We put node1 in "loadbalancer" group and node2 with node3 in "backend" group.

Here is our `inventory.ini`:

```ini
[loadbalancer]
node1 ansible_host=node1.smartm2m.co.kr

[backend]
node2 ansible_host=node2.smartm2m.co.kr
node3 ansible_host=node3.smartm2m.co.kr
```

### Step 2: Install Nginx on Servers

Next, we make playbook to install Nginx on all servers. This is our `start-nginx.yml` playbook:

```yml
---
- name: Install Nginx on all servers
  hosts: all
  become: yes

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: latest
        update_cache: yes
      notify: start nginx

  handlers:
    - name: start nginx
      service:
        name: nginx
        state: started
        enabled: yes
```

Run this command to install:

```bash
ansible-playbook -i inventory.ini start-nginx.yml
```

### Step 3: Update Nginx Index Page on Backend

Now we change Nginx default page. We make it show "served from: <hostname>" on backend servers. This is the `update-nginx.yml` playbook:

```yml
---
- name: Update Nginx index page on backend servers
  hosts: backend
  become: yes

  tasks:
    - name: Create index.html with the hostname
      template:
        src: index.html.j2
        dest: /usr/share/nginx/html/index.html
      notify: reload nginx

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

This is the `index.html.j2`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Welcome to {{ ansible_hostname }}</title>
</head>
<body>
    <h1>Served from: {{ ansible_hostname }}</h1>
</body>
</html>
```

Use this command to change the page:

```bash
ansible-playbook -i inventory.ini update-nginx.yml
```

### Step 4: Set Up Load Balancing

Last, we make playbook for load balancing. We use "least connection" method. Here is `setup-loadbalancer.yml` playbook:

```yml
---
- hosts: loadbalancer
  become: true
  vars:
    backend_servers:
      - host: node2
        weight: 1
      - host: node3
        weight: 1
  tasks:
    - name: Setup Nginx as Load Balancer
      template:
        src: loadbalancer.conf.j2
        dest: /etc/nginx/conf.d/loadbalancer.conf
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

This is the `loadbalancer.conf.j2` config:

```conf
upstream backend {
    least_conn;
    {% for server in backend_servers %}
    server {{ server.host }}:80 weight={{ server.weight }};
    {% endfor %}
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
    }
}
```

Run this to set up load balancing:

```bash
ansible-playbook -i inventory.ini setup-loadbalancer.yml
```

### Ansible Conclusion

We did it. We set up Nginx on all servers, made custom page for backend, and made load balancer. Now, backend servers show page with their hostname and load balancer shares requests between backend servers by "least connection" method.

## Kubernetes
1. Create a deployment as follows:
- Name: nginx-app
- Using container nginx with version 1.11.10-alpine
- The deployment should contain 3 replicas
2. Next, deploy the application with new version 1.11.13-alpine, by performing a rolling update.
Finally, rollback that update to the previous version 1.11.10-alpine.
3. Set a node named k8s-node-1 as unavailable and reschedule all the pods running on it.
4. Create a Persistent Volume with name app-data, of capacity 2Gi and access mode
ReadWriteMany. The type of volume is hostPath and its location is /srv/app-data.
5. Create a Persistent Volume Claim that requests the Persistent Volume you had created
above. The claim should request 2Gi. Ensure that the Persistent Volume Claim has the
same storageClassName as the Persistent Volume you had previously created.
6. You have been asked to set up a Kubernetes cluster, one master and one worker node.
You have done the initialization of the master, what is the next steps to make the worker
node join the cluster?

### Step 1: Create a Deployment

We create a Kubernetes deployment named `nginx-app` using the `nginx:1.11.10-alpine` container image, with 3 replicas.

Here is the `nginx-deployment.yml` manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.11.10-alpine
```

Apply this deployment with the command:

```bash
kubectl apply -f nginx-deployment.yml
```

### Step 2: Rolling Update and Rollback

To update the application to version `1.11.13-alpine`, we'll perform a rolling update.

```bash
kubectl set image deployment/nginx-app nginx=nginx:1.11.13-alpine --record
```

To rollback to the previous version (`1.11.10-alpine`), use the following command:

```bash
kubectl rollout undo deployment/nginx-app
```

### Step 3: Node Maintenance

To set the node `k8s-node-1` as unavailable and reschedule the pods:

```bash
kubectl drain k8s-node-1 --ignore-daemonsets --delete-local-data
```

### Step 4: Create a Persistent Volume

Here is the manifest for creating a Persistent Volume named `app-data`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-data
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/srv/app-data"
  storageClassName: manual
```

Apply this Persistent Volume with the command:

```bash
kubectl apply -f persistent-volume.yml
```

### Step 5: Create a Persistent Volume Claim

We create a Persistent Volume Claim to request the Persistent Volume created above.

Here is the `persistent-volume-claim.yml` manifest:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-claim
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  storageClassName: manual
```

Apply this Persistent Volume Claim with the command:

```bash
kubectl apply -f persistent-volume-claim.yml
```

### Step 6: Join Worker Node to the Cluster

After initializing the master node, the next step is to join the worker node to the cluster.

1. On the master node, retrieve the join command:

```bash
kubeadm token create --print-join-command
```

2. Run the join command on the worker node:

```bash
kubeadm join [Master node IP]:6443 --token [token] --discovery-token-ca-cert-hash sha256:[hash]
```

Make sure to replace `[Master node IP]`, `[token]`, and `[hash]` with the actual values provided by the master node's output.

Now the worker node should be part of our Kubernetes cluster.
After running the command, we can check the status of our nodes in the cluster by running:
```bash
kubectl get nodes
```

### Kubernetes Conclusion

With the steps above, we've managed to set up Nginx on Kubernetes with rolling updates and rollbacks, managed node availability, set up persistent storage with the same storage class, and expanded our cluster with a worker node.