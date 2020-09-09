## Creating WordPress Helm Chart

In this project, shown how to create a WordPress helm chart.

### Requirements

Kubernetes Cluster

Helm v3.3.0

## WordPress Helm Chart

1. First, create a directory for the chart.
```
mkdir wordpress_chart/
mkdir wordpress_chart/templates 
```

2. Create Chart.yaml file in ``wordpress_chart/`` directory.
```
cat <<EOF >Chart.yaml
apiVersion: v1
name: Wordpress
version: 0.1.0
appVersion: 1
description: First created chart. 
EOF
```

3. Create deployment yaml file for mysql.
```
cat <<EOF >wordpress_chart/templates/mysql-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: mysql
  name: mysql
spec:
  replicas: {{ .Values.replicas.mysql  }}
  selector:
    matchLabels:
      app: mysql
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: mysql
    spec:
      containers:
      - image: {{ .Values.image.mysql  }}
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-cluster-secrets
              key: root
  
        resources: {}
status: {}
EOF
```

4. Create service yaml file for mysql.
```
kubectl expose deployment mysql --port 3306 --dry-run -o yaml > wordpress_chart/templates/mysql-svc.yaml
```

5. Create deployment yaml file for WordPress.
```
cat <<EOF > wordpress_chart/templates/wp-deploy.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: {{ .Values.replicas.wordpress  }}
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - wordpress
            topologyKey: "kubernetes.io/hostname"
      tolerations:
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 20
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 20
      volumes:
      - name: wp-volume
        persistentVolumeClaim:
          claimName: wp-claim        
      containers:
      - image: {{ .Values.image.wordpress  }}
        name: wordpress
        volumeMounts:
        - name: wp-volume
          mountPath: /var/www/html/
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-cluster-secrets
              key: root
        ports:
        - containerPort: 80
          name: wordpress
EOF
```

6. Create service yaml file for WordPress.
```
kubectl expose deployment wordpress --port 80 --type=NodePort --dry-run -o yaml > wordpress_chart/templates/wp-svc.yaml
```

7. Create a secret for mysql root password.
```
cat <<EOF > wordpress_chart/templates/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-cluster-secrets
type: Opaque
data:
  root: {{ .Values.password.root  }}
EOF
```

8. Create values.yaml for variables.
```
cat <<EOF >wordpress_chart/values.yaml
replicas:
  mysql: 1
  wordpress: 1

image:
  wordpress: wordpress:latest
  mysql: mysql:latest


# The following command can be used to get base64-encoded password from a plain text string: $ echo -n 'plain-text-password' | base64
# Fill mysql root password
password:
  root: cm9vdF9wYXNzd29yZA==
EOF
```

For testing this helm chart you need to create 1 persistent volume. After that you can run your chart by executing following command inside ``wordpress_chart/`` directory.

```
helm install wordpress .
```
