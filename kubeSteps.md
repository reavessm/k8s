# Kube Cluster

This is the steps to recreate my kubernetes cluster.  I will probably make a `kubeSteps.yaml` to accompany this

## Step 1

Install

* Gentoo
* Kubeadm
* Docker
* Kernel config
* virt-manager

## Step 2

[Galera](https://severalnines.com/blog/mysql-docker-running-galera-cluster-kubernetes)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: datadir-galera-0
  labels:
    app: galera-ss
    podindex: "0"
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  hostPath:
    path: /data/pods/galera-0/datadir
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-datadir-galera-ss-0
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  selector:
    matchLabels:
      app: galera-rs
---
apiVersion: v1
kind: Service
metadata:
  name: galera-rs
  labels:
    app: galera-rs
spec:
  type: NodePort
  ports:
    - nodePort: 30000
      port: 3306
  selector:
    app: galera

---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: galera
  labels:
    app: galera
spec:
  replicas: 3
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: galera
    spec:
      containers:
      - name: galera
        image: severalnines/mariadb:10.1
        env:
        # kubectl create secret generic mysql-pass --from-file=password.txt
        - name: MYSQL_ROOT_PASSWORD
          value: myrootpassword
        - name: DISCOVERY_SERVICE
          value: etcd-client:2379
        - name: XTRABACKUP_PASSWORD
          value: password
        - name: CLUSTER_NAME
          value: mariadb_galera
        - name: MYSQL_DATABASE
          value: mydatabase
        - name: MYSQL_USER
          value: myuser
        - name: MYSQL_PASSWORD
          value: myuserpassword
        ports:
        - name: mysql
          containerPort: 3306
        readinessProbe:
          exec:
            command:
            - /healthcheck.sh
            - --readiness
          initialDelaySeconds: 120
          periodSeconds: 1
        livenessProbe:
          exec:
            command:
            - /healthcheck.sh
            - --liveness
          initialDelaySeconds: 120
          periodSeconds: 1
```

## Step 2

[KeyCloak](https://dzone.com/articles/setup-keycloak-cluster-with-kube-ping-in-kubernete)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  labels:
    app: keycloak
    name: keycloak
spec:
  type: NodePort
  ports:
    - name: http
      protocol: TCP
      port: 8080
      nodePort: 30000
  selector:
    app: keycloak
    name: keycloak
---
apiVersion: v1
items:
- apiVersion: extensions/v1beta1
  kind: Deployment
  spec:
    replicas: 3
    template:
       metadata:
         labels:
           app: keycloak
           name: keycloak
       spec:
        containers:
        - env:
          - name: KEYCLOAK_HOSTNAME
            value: {{keycloak host name}}
          - name: KEYCLOAK_LOGLEVEL
            value: DEBUG
          - name: ROOT_LOGLEVEL
            value: DEBUG
          - name: KEYCLOAK_USER
            value: {{keyclock admin user}}
          - name: DB_VENDOR
            value: mysql
          - name: DB_ADDR
            value: {{mysql host}}
          - name: DB_USER
            value: {{mysql user}}
          - name: DB_PASSWORD
            value: {{mysql password}}
          - name: JGROUPS_DISCOVERY_PROTOCOL
            value: kubernetes.KUBE_PING
          - name: JGROUPS_DISCOVERY_PROPERTIES
            value: port_range=0,dump_requests=true
          - name: connectTimeout
            value: "600000"
          - name: KEYCLOAK_PASSWORD
            value: {{keyclock admin password}}
          - name: remoteTimeout
            value: "600000"
          image: jboss/keycloak:4.5.0
          imagePullPolicy: Never
          name: keycloak
          ports:
          - containerPort: 8080
            name: http
            protocol: TCP
          - containerPort: 8443
            name: https
            protocol: TCP
```

## Step 3

Nginx-LetsEncrypt Reverse Proxy

Config-Map

Secrets

Does this need to be replicated?  I mean why not??
