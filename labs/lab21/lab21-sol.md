---
layout: lab
title: "Práctica 21 (Instructor): Llaves / Soluciones — NO PUBLICAR"
permalink: /lab21/lab21-instructor/
images_base: /labs/lab21/img
duration: "120 minutos (sugerido)"
objective:
  - "Guía de resolución (llaves) para el mini-examen estilo CKAD. **NO PUBLICAR** a alumnos."
slug: lab21-instructor
lab_number: 21
notes:
  - "Esta guía contiene soluciones completas. Mantener privada."
---

> **IMPORTANTE:** Esta sección es la **llave**. Si vas a entregar el reto a alumnos, **NO incluyas este documento** en el sitio público.

---

## Setup rápido (Reto 1)

```bash
kubectl create ns ckad-reto
kubectl config set-context --current --namespace=ckad-reto
alias k=kubectl
```

---

## Reto 2 (imagen + deploy) — dos caminos

### Opción A (recomendada para examen/entorno sin registry): Nginx + ConfigMap (sin docker push)

> Esto no cumple “docker build” pero es ideal cuando no hay acceso a un registry. Úsalo como plan B.

```bash
kubectl create configmap web-content --from-literal=index.html="<h1>CKAD Reto WebApp</h1>" --dry-run=client -o yaml > manifests/02a-web-content-cm.yaml
```
```bash
kubectl apply -f manifests/02a-web-content-cm.yaml
```

Deployment ejemplo (monta ConfigMap en html). Ajusta si ya existe un deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: ckad-reto
  labels: { app: webapp }
spec:
  replicas: 2
  selector:
    matchLabels: { app: webapp }
  template:
    metadata:
      labels: { app: webapp }
    spec:
      containers:
      - name: webapp
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: web-content
          items:
          - key: index.html
            path: index.html
```
```bash
kubectl apply -f manifests/02b-deployment.yaml
```

### Opción B (con docker build + push)

```bash
mkdir -p app
cat > app/Dockerfile <<'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EOF

cat > app/index.html <<'EOF'
<h1>CKAD Reto WebApp</h1>
EOF

# build/push según tu registry (ECR o Docker Hub)
docker build -t <tu-registry>/ckad-webapp:v1 app
docker push <tu-registry>/ckad-webapp:v1
```

Deployment + Service (generación rápida):

```bash
kubectl create deploy webapp --image=<tu-registry>/ckad-webapp:v1 --replicas=2 --dry-run=client -o yaml > manifests/02-webapp-deploy.yaml
kubectl apply -f manifests/02-webapp-deploy.yaml
```

---

## Reto 3 Service + prueba de conectividad

```bash
kubectl expose deploy webapp --name webapp-svc --port 80 --target-port 80 --dry-run=client -o yaml > manifests/03-webapp-svc.yaml
```
```bash
kubectl apply -f manifests/03-webapp-svc.yaml
```

---

## Reto 4 (ConfigMap/Secret)

```bash
kubectl create cm app-config --from-literal=APP_COLOR=blue --from-literal=APP_TITLE=CKAD-Reto --dry-run=client -o yaml > manifests/04-app-config.yaml
```
```bash
kubectl apply -f manifests/04-app-config.yaml
```
```bash
kubectl create secret generic app-secret --from-literal=API_KEY=supersecret --dry-run=client -o yaml > manifests/04-app-secret.yaml
```
```bash
kubectl apply -f manifests/04-app-secret.yaml
```

Parche sugerido al deployment (env + secret volume):

```yaml
spec:
  template:
    spec:
      containers:
      - name: webapp
        envFrom:
        - configMapRef:
            name: app-config
        volumeMounts:
        - name: secretvol
          mountPath: /etc/secret
          readOnly: true
      volumes:
      - name: secretvol
        secret:
          secretName: app-secret
          items:
          - key: API_KEY
            path: API_KEY
```
```bash
kubectl apply -f manifests/02b-deployment.yaml
```

---

## Reto 5 (probes)

Debe ir al mismo nivel que image, ports, envFrom, volumeMounts, etc.

```yaml
  readinessProbe:
    httpGet:
      path: /
      port: 80
    initialDelaySeconds: 3
    periodSeconds: 5

  livenessProbe:
    httpGet:
      path: /
      port: 80
    initialDelaySeconds: 10
    periodSeconds: 10
```
```bash
kubectl apply -f manifests/02b-deployment.yaml
```

Fallo controlado: cambiar temporalmente `path: /bad` en readiness o liveness y observar.

---

## Reto 6 (rollout/rollback)

```bash
# (opcional) ver imagen actual
kubectl -n ckad-reto get deploy webapp -o=jsonpath='{.spec.template.spec.containers[0].image}'; echo
```
```bash
# 1) Rolling update a "v2" (tag pública)
kubectl -n ckad-reto set image deploy/webapp webapp=nginx:1.26-alpine --record
kubectl -n ckad-reto rollout status deploy/webapp
```
```bash
# 2) Ver history
kubectl -n ckad-reto rollout history deploy/webapp
```
```bash
# 3) Simular fallo (imagen que NO existe)
kubectl -n ckad-reto set image deploy/webapp webapp=nginx:0.0.0-does-not-exist --record
kubectl -n ckad-reto rollout status deploy/webapp || true
```
```bash
# (evidencia rápida del fallo)
kubectl -n ckad-reto get pods -l app=webapp
kubectl -n ckad-reto describe rs -l app=webapp | egrep -n "Image|Failed|ErrImage|Back-off|Pull"
```
```bash
# 4) Rollback
kubectl -n ckad-reto rollout undo deploy/webapp
kubectl -n ckad-reto rollout status deploy/webapp
```
```bash
# 5) Confirmar imagen final (debe volver a la anterior)
kubectl -n ckad-reto get deploy webapp -o=jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

---

## Reto 7 (sidecar)

`manifests/07-sidecar-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
  namespace: ckad-reto
  labels:
    app: sidecar-demo
spec:
  terminationGracePeriodSeconds: 1
  volumes:
  - name: logs
    emptyDir: {}
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh","-c"]
    args:
      - |
        mkdir -p /var/log;
        i=0;
        while true; do
          i=$((i+1));
          echo "$(date -Iseconds) msg $i" >> /var/log/app.log;
          sleep 2;
        done
    resources:
      requests:
        cpu: 5m
        memory: 16Mi
      limits:
        cpu: 50m
        memory: 64Mi
    volumeMounts:
    - name: logs
      mountPath: /var/log

  - name: sidecar
    image: busybox:1.36
    command: ["sh","-c"]
    args:
      - |
        # espera a que exista el archivo, luego tailea
        until [ -f /var/log/app.log ]; do sleep 1; done;
        tail -n+1 -F /var/log/app.log
    resources:
      requests:
        cpu: 5m
        memory: 16Mi
      limits:
        cpu: 50m
        memory: 64Mi
    volumeMounts:
    - name: logs
      mountPath: /var/log
      readOnly: true
```
```bash
kubectl -n ckad-reto apply -f manifests/07-sidecar-pod.yaml
kubectl -n ckad-reto get pod -l app=sidecar-demo -w
```

---

## Reto 8 (Ingress)

`manifests/08-webapp-ingress.yaml`:

Incluye ingressClassName (si tu clúster tiene NGINX, suele ser nginx; en EKS con AWS Load Balancer Controller suele ser alb).

## NGINX CONTROLLER:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```
```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ing
  namespace: ckad-reto
spec:
  ingressClassName: nginx
  rules:
  - host: webapp.ckad.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
```

```bash
kubectl -n ckad-reto apply -f manifests/08-webapp-ingress.yaml
```
```bash
kubectl -n ckad-reto get ingress
```
```bash
kubectl -n ckad-reto describe ingress webapp-ing
```
```bash
kubectl -n ingress-nginx port-forward svc/ingress-nginx-controller 8081:80
```
```bash
curl -s -H "Host: webapp.ckad.local" http://127.0.0.1:8081/ | head -n 20
```
LB
```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide | tee outputs/08_ingressnginx_svc.txt
ADDR="$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{.status.loadBalancer.ingress[0].ip}')"
echo "$ADDR" | tee outputs/08_ingress_lb_address.txt
curl -s -H "Host: webapp.ckad.local" "http://$ADDR/" | head
```

---

## Reto 9 (NetworkPolicy)

activar el addon network policy

```bash
aws eks update-addon \
  --region "us-west-2" \
  --cluster-name "sim-exam-eks" \
  --addon-name vpc-cni \
  --resolve-conflicts PRESERVE \
  --configuration-values '{"enableNetworkPolicy":"true"}'
```
```bash
kubectl -n kube-system rollout status ds/aws-node 
kubectl -n kube-system get pods -l k8s-app=aws-node -o wide
```
```bash
kubectl get crd | grep -i policyendpoints
kubectl -n ckad-reto get policyendpoints 2>/dev/null
```
```bash
kubectl -n ckad-reto rollout restart deploy/webapp
kubectl -n ckad-reto rollout status deploy/webapp
```

Default deny ingress para `app=webapp`: `manifests/09-webapp-deny-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: webapp-deny-ingress
  namespace: ckad-reto
spec:
  podSelector:
    matchLabels:
      app: webapp
  policyTypes:
  - Ingress
```

Allow solo desde `role=tester` al puerto 80: `manifests/09-webapp-allow-from-tester.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: webapp-allow-from-tester
  namespace: ckad-reto
spec:
  podSelector:
    matchLabels:
      app: webapp
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: tester
    ports:
    - protocol: TCP
      port: 80
```
```bash
kubectl -n ckad-reto apply -f manifests/09-webapp-deny-ingress.yaml
kubectl -n ckad-reto apply -f manifests/09-webapp-allow-from-tester.yaml
```
```bash
kubectl -n ckad-reto get networkpolicy
```
```bash
kubectl -n ckad-reto describe networkpolicy webapp-deny-ingress
```
```bash
kubectl -n ckad-reto describe networkpolicy webapp-allow-from-tester
```


---

## Reto 10 (Job/CronJob)

Job `pi`: `manifests/10-job-cronjob.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
  namespace: ckad-reto
spec:
  backoffLimit: 1
  ttlSecondsAfterFinished: 120
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: pi
        image: busybox:1.36
        command: ["sh","-c","echo pi-ok; sleep 2; echo done"]
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: heartbeat
  namespace: ckad-reto
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 0
      ttlSecondsAfterFinished: 120
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: tick
            image: busybox:1.36
            command: ["sh","-c","date; echo tick"]
```
```bash
kubectl -n ckad-reto apply -f manifests/10-job-cronjob.yaml
```
```bash
kubectl -n ckad-reto logs job/pi --tail=5
```


---

## Reto 11 (PVC)

Habilita OIDC (para IRSA)

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster "sim-exam-eks" \
  --region "us-west-2" \
  --approve
```

Crea el ServiceAccount con permisos (AmazonEBSCSIDriverPolicy)

```bash
eksctl create iamserviceaccount \
  --cluster "sim-exam-eks" \
  --region "us-west-2" \
  --namespace kube-system \
  --name ebs-csi-controller-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts
```

Instalar el Addon driver.

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
```
``bash
helm upgrade --install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  -n kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=ebs-csi-controller-sa
```

Verifica que ya estén los Pods del CSI:

```bash
kubectl -n kube-system get pods | egrep -i 'ebs|csi'
```

Verificar Storage Class

```bash
kubectl get storageclass -o wide
kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}'
```

PVC: `manifests/11-pvc-writer.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: ckad-reto
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: gp2
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pvc-writer
  namespace: ckad-reto
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
  containers:
  - name: writer
    image: busybox:1.36
    command: ["sh","-c"]
    args:
      - |
        set -e
        echo "hello" > /data/hello.txt
        echo "wrote at: $(date -Iseconds)" >> /data/hello.txt
        sleep 3600
    volumeMounts:
    - name: data
      mountPath: /data
```
```bash
kubectl -n ckad-reto apply -f manifests/11-pvc-writer.yaml
```
```bash
kubectl -n ckad-reto get pvc data-pvc
```
```bash
kubectl -n ckad-reto describe pvc data-pvc
```
```bash
kubectl -n ckad-reto get pod pvc-writer -o wide
```
```bash
kubectl -n ckad-reto exec pvc-writer -- sh -c 'ls -la /data && cat /data/hello.txt'
```
```bash
kubectl -n ckad-reto delete pod pvc-writer
kubectl -n ckad-reto apply -f manifests/11-pvc-writer.yaml
kubectl -n ckad-reto exec pvc-writer -- sh -c 'cat /data/hello.txt'
```


---

## Reto 12 (RBAC)

Crear el archivo: `manifests/12-rbac-app-sa.yaml`
Role (pods + pods/log):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: ckad-reto
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-reader
  namespace: ckad-reto
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-binding
  namespace: ckad-reto
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: ckad-reto
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-reader
```
```bash
kubectl apply -f manifests/12-rbac-app-sa.yaml
```
Validación:

```bash
kubectl -n ckad-reto auth can-i list pods --as=system:serviceaccount:ckad-reto:app-sa
kubectl -n ckad-reto auth can-i delete pods --as=system:serviceaccount:ckad-reto:app-sa
kubectl -n ckad-reto auth can-i get pods --subresource=log --as=system:serviceaccount:ckad-reto:app-sa
```
```bash
kubectl -n ckad-reto auth can-i get secrets --as=system:serviceaccount:ckad-reto:app-sa
kubectl -n ckad-reto auth can-i create deployments --as=system:serviceaccount:ckad-reto:app-sa
```

