Your company’s java-mysql application is running with docker-compose on a server. This application is used often internally and by your company clients too. You noticed that the server isn't very stable: Often a database container dies or the application itself, or docker daemon must be restarted. During this time people can't access the app!

So when this happens, the users write to you to tell you that the app is down and ask you to fix it. You SSH into the server, restart the containers with docker-compose and containers start again.

But this is annoying work, plus it doesn't look good for your company that your clients often can't access the app. So you want to make your application more reliable and highly available. You want to replicate both the database and the app, so if one container goes down, there is always a backup. Also you don't want to rely on a single server, but have multiple, in case 1 whole server goes down or gets rebooted etc.



So you look into different solutions and decide to use the container orchestration tool Kubernetes to solve the issue. For now you want to configure it and deploy your application manually, since it's a new tool and want to try it out manually before automating.

---

# Deploying Java-MySQL Application on Kubernetes

This guide will walk you through the process of deploying a Java-MySQL application on Kubernetes, making it more reliable and highly available.

## Step 1: Create a Kubernetes cluster

Create a Kubernetes cluster using either Minikube or Linode Kubernetes Engine (LKE).

**Minikube**

```sh
minikube start
```

**LKE**

```sh
On Linode UI dashboard, create K8s cluster with 2 smallest nodes "Dedicated 4 GB" plan
Update your KUBECONFIG context to connect to Linode environment
```

## Step 2: Deploy MySQL with 2 replicas

Deploy MySQL database with 2 replicas and volumes for data persistence. Helm will be used to simplify the process.

- Mysql Chart link: 
https://github.com/bitnami/charts/tree/master/bitnami/mysql 

**Minikube**
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-release bitnami/mysql -f mysql-chart-values-minikube.yaml

```

**LKE**
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-release bitnami/mysql -f mysql-chart-values-lke.yaml

```


## Step 3: Deploy your Java Application with 2 replicas

Deploy the Java application with 2 replicas. The image location is `nanatwn/demo-app:mysql-app`. Set up ConfigMap and Secret with the correct values and reference them in the application deployment config file.


**Minikube & LKE**

```sh
# Create my-registry-key secret to pull image
DOCKER_REGISTRY_SERVER=docker.io
DOCKER_USER=your dockerID, same as for `docker login`
DOCKER_EMAIL=your dockerhub email, same as for `docker login`
DOCKER_PASSWORD=your dockerhub pwd, same as for `docker login`

kubectl create secret docker-registry my-registry-key \
--docker-server=$DOCKER_REGISTRY_SERVER \
--docker-username=$DOCKER_USER \
--docker-password=$DOCKER_PASSWORD \
--docker-email=$DOCKER_EMAIL


kubectl apply -f db-secret.yaml
kubectl apply -f db-config.yaml
kubectl apply -f java-app.yaml

```


## Step 4: Deploy phpMyAdmin

Deploy phpMyAdmin to access the MySQL UI. This deployment requires only 1 replica since it's for internal use.


**Minikube & LKE**

```sh
kubectl apply -f phpmyadmin.yaml

```

---
Now your application setup is running in the cluster, but you still need a proper way to access the application. Also, you don't want users to access the application using the IP address but instead to use a domain name. For that, you want to install Ingress controller in the cluster and configure ingress access for your application.

---

## Step 5: Deploy Ingress Controller

Deploy an Ingress Controller in the cluster using Helm to properly access the application.

**Minikube**

```sh
# minikube comes with ingress addon, so we just need to activate it
minikube addons enable ingress 

```

**LKE**

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx

```

**Notes on installing Ingress-controller on LKE**
- Chart link: https://github.com/kubernetes/ingress-nginx/tree/main/charts/ingress-nginx


<img src="https://i.imgur.com/Q6akU4k.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


----


## Step 6: Create Ingress rule

Create an Ingress rule for your application's access. For Minikube, the application must be accessible on `my-java-app.com`. For LKE, update line 48 of the Index.html file for the application to include the Linode node-balancer address.

**Minikube**

- set the host name in java-app-ingress.yaml line 7 to my-java-app.com
- add `127.0.0.1 my-java-app.com` in /etc/hosts file
- create ingress component: `kubectl apply -f java-app-ingress.yaml`
- run `minikube tunnel` command in terminal window
- access application from browser on address: `my-java-app.com`

**LKE**
- set the HOST variable found at line 48 of the index.html to the Linode node-balancer address
- set the host name in java-app-ingress.yaml line 7 to Linode node-balancer address
- create ingress component: `kubectl apply -f java-app-ingress.yaml`
- access application from browser on Linode node-balancer address


<img src="https://i.imgur.com/UqigaU1.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

<img src="https://i.imgur.com/YmzZEwX.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

<img src="https://i.imgur.com/RMCnghA.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---


## Step 7: Port-forward for phpMyAdmin

Configure port-forwarding for phpMyAdmin to access it securely on localhost whenever needed.

**Minikube & LKE**
```sh
kubectl port-forward svc/phpmyadmin-service 8081:8081

```

---

As the final step, you decide to create a helm chart for your Java application where all the configuration files are configurable. You can then tell developers how they can use it by setting all the chart values. This chart will be hosted in its own git repository.

---


## Step 8: Create Helm Chart for Java App

Create a Helm Chart for your Java application including service, deployment, ingress, ConfigMap, and Secret. Provide a custom values file as an example for developers to use when deploying the application. Host the chart in its own git repository.

- create helm chart boilerplate for your application with chart-name `java-app` using command: `helm create java-app`

***Note**: This will generate `java-app` folder with chart files*

- clean up all unneeded contents from `java-app` folder, as you learned in the module
- create template files for `db-config.yaml`, `db-secret.yaml`, `java-app-deployment.yaml`, `java-app-ingress.yaml`, `java-app-service.yaml`
- create `values-override.yaml` and set all the correct values there 
- set default chart values in `values.yaml` file


***Note**: the `ingress.hostName` must be set to `my-java-app.com` for Minikube & Linode node balancer address*

- validate that your chart is correct and debug any issues, do a dry-run

`helm install my-cool-java-app java-app -f java-app/values-override.yaml --dry-run --debug`

- if dry-run shows the k8s manifest files with correct values, everything is working, so you can create the chart release

`helm install my-cool-java-app java-app -f java-app/values-override.yaml` 

- extract the chart `java-app` folder and host into its own new git repository `java-app-chart` 

---

<img src="https://i.imgur.com/QfVNEqw.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

<img src="https://i.imgur.com/rRE7L1O.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

**db-config.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Values.configName }}
data:
  {{- range $key, $value := .Values.configData}}
  {{ $key }}: {{ $value }}
  {{- end}}
```

---

**db-secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.secretName }}
type: Opaque
data:
  {{- range $key, $value := .Values.secretData}}
  {{ $key }}: {{ $value | b64enc }}
  {{- end}}
```

---

**java-app-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}-deployment
  labels:
    app: {{ .Values.appName }}
spec:
  replicas: {{ .Values.appReplicas }}
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
    spec:
      imagePullSecrets:
      - name: {{ .Values.registrySecret }}
      containers:
      - name: {{ .Values.appContainerName }}
        image: "{{ .Values.appImage }}:{{ .Values.appVersion }}"
        ports:
        - containerPort: {{ .Values.containerPort }}
        env:
        {{- range $key, $value := .Values.regularData }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end}}

        {{- range $key, $value := .Values.secretData}}
        - name: {{ $key }}
          valueFrom:
            secretKeyRef:
              # in loop, we lose global context, but can access global context with $
              # $ is 1 variable that is always global and will always point to the root context
              # so $.Value instead of .Values
              name: {{ $.Values.secretName }}
              key: {{ $key }}
        {{- end}}

        {{- range $key, $value := .Values.configData}}
        - name: {{ $key }}
          valueFrom:
            configMapKeyRef:
              name: {{ $.Values.configName }}
              key: {{ $key }}
        {{- end}}
```

---

**java-app-ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Values.appName }}-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: {{ .Values.ingress.hostName }}
    http:
      paths:
      - backend:
          service:
            name: {{ .Values.appName }}
            port: 
              number: {{ .Values.servicePort }}
        pathType: {{ .Values.ingress.pathType }}
        path: {{ .Values.ingress.path }}
```

---

**java-app-service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.appName }}-service
spec:
  selector:
    app: {{ .Values.appName }}
  ports:
  - protocol: TCP
    port: {{ .Values.servicePort }}
    targetPort: {{ .Values.containerPort }}
```

---

**values-override.yaml**

```yaml
appName: java-app
appReplicas: 3
registrySecret: my-registry-key
appContainerName: javamysqlapp
appImage: nanajanashia/demo-app
appVersion: java-mysql-app
containerPort: 8080

servicePort: 8080

configName: db-config
configData:
  DB_SERVER: my-release-mysql-primary

secretName: db-secret
secretData: 
  DB_USER: my-user
  DB_PWD: my-pass
  DB_NAME: my-app-db
  MYSQL_ROOT_PASSWORD: secret-root-pass

regularData: {}
 # MY_ENV: my-value

ingress:
  hostName: my-java-app.com # set this value to Linode nodebalancer address for LKE
  pathType: Exact
  path: /
```



