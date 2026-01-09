---
layout: lab
title: "Práctica 16: Configuración avanzada de RBAC en EKS"
permalink: /lab16/lab16/
images_base: /labs/lab16/img
duration: "60 minutos"
objective:
  - "Implementar un modelo RBAC avanzado en Amazon EKS separando responsabilidades por namespace (dev/prod), aplicando **least privilege**, controlando acciones sensibles (**pods/log** y **pods/exec** como permisos separados), restringiendo cambios con **resourceNames**, y validando permisos de forma práctica con **kubectl auth can-i** e **impersonation**."
prerequisites:
  - "Windows + Visual Studio Code + Terminal GitBash"
  - "Herramientas: kubectl (obligatorio); AWS CLI v2 (recomendado); eksctl (recomendado para creación de clúster); jq (opcional)"
  - "Permisos Kubernetes: idealmente **cluster-admin** (o mínimo: crear Namespace, Role/RoleBinding, ServiceAccount, Deployment/Service)"
  - "Permisos AWS: para crear/consultar EKS (si crearás el clúster en esta práctica)"
  - "Importante: para usar `--as/--as-group` (impersonation), tu identidad debe tener permiso **impersonate**. En labs normalmente lo tienes si eres admin."
introduction:
  - "RBAC (Role-Based Access Control) autoriza acciones en Kubernetes a partir de **Roles/ClusterRoles** (qué se puede hacer) y **RoleBindings/ClusterRoleBindings** (a quién se le asigna). En EKS, IAM resuelve la **autenticación** (quién eres) y RBAC decide la **autorización** (qué puedes hacer). En esta práctica separarás permisos entre **dev** y **prod**, endurecerás acciones de alto riesgo (logs/exec), y comprobarás resultados con `kubectl auth can-i` como si fuera un “unit test” de seguridad."
slug: lab16
lab_number: 16
final_result: >
  Al finalizar tendrás un esquema RBAC avanzado en EKS con separación dev/prod, identidades (ServiceAccounts) con permisos mínimos por rol (despliegue, observabilidad, debug), controles finos por subrecurso (pods/log y pods/exec) y restricciones por objeto (resourceNames), validado con pruebas positivas/negativas y troubleshooting con describe/auth can-i/impersonation al estilo CKAD.
notes:
  - "CKAD: práctica altamente relevante (RBAC, subrecursos, troubleshooting con `kubectl auth can-i`, lectura de Roles/RoleBindings, y pruebas negativas/positivas)."
  - "No existe un “deny” explícito en RBAC: la seguridad se logra **no otorgando** permisos."
  - "Hardening recomendado: evita conceder `secrets` y `pods/exec` por defecto; concédelos solo a quien lo necesite."
  - "La creación del clúster EKS se incluye en esta práctica para no asumir preexistencia. Si ya tienes un clúster listo, reutilízalo y **omite** la Tarea 2."
references:
  - text: "Kubernetes - Using RBAC Authorization (resourceNames, subresources)"
    url: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
  - text: "Kubernetes - Authorization overview (incluye `kubectl auth can-i`)"
    url: https://kubernetes.io/docs/reference/access-authn-authz/authorization/
  - text: "Kubernetes - User impersonation (concepto y consideraciones)"
    url: https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation
  - text: "Amazon EKS Best Practices - Identity and Access Management (Access Entries vs aws-auth)"
    url: https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html
  - text: "eksctl - Getting started / create cluster"
    url: https://eksctl.io/
prev: /lab15/lab15/
next: /lab17/lab17/
---

---

### Tarea 1. Preparación del workspace y baseline (herramientas + contexto)

Crearás la carpeta del laboratorio, definirás variables reutilizables y validarás herramientas, identidad y conectividad al clúster (o confirmarás que debes crearlo).

> **NOTA (CKAD):** Muchísimos fallos se explican con: contexto incorrecto, permisos, y eventos. Practica `kubectl describe`, `kubectl get events` y `kubectl auth can-i`.
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo del curso con un usuario con permisos administrativos.

- {% include step_label.html %} Abre **Visual Studio Code** y la terminal integrada.

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Git Bash** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar en la carpeta raíz de tus laboratorios (por ejemplo **`labs-eks-ckad`**).

  > **NOTA:** Si vienes de otra práctica, usa `cd ..` hasta volver a la raíz.
  {: .lab-note .info .compact}

- {% include step_label.html %} Crea el directorio del laboratorio y la estructura estándar.

  ```bash
  mkdir -p lab16 && cd lab16
  mkdir -p k8s rbac outputs
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que la estructura quedó creada (evidencia).

  ```bash
  find . -maxdepth 2 -type d | sort | tee outputs/00_dirs.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables base (ajusta a tu entorno).

  ```bash
  export AWS_REGION="us-west-2"
  export CLUSTER_NAME="eks-lab16-rbac"
  export K8S_VERSION="1.33"

  export NS_DEV="rbac-dev"
  export NS_PROD="rbac-prod"
  ```

- {% include step_label.html %} Verifica CLIs (evidencia de herramientas disponibles).

  ```bash
  kubectl version --client=true | tee outputs/01_kubectl_version.txt
  aws --version 2>&1 | tee outputs/01_aws_version.txt || true
  eksctl version 2>&1 | tee outputs/01_eksctl_version.txt || true
  jq --version 2>&1 | tee outputs/01_jq_version.txt || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica tu identidad en AWS (si falla, corrige credenciales antes de continuar).

  ```bash
  aws sts get-caller-identity | tee outputs/01_aws_identity.json
  ```
  {% include step_image.html %}
  ```bash
  aws configure get region || true
  echo "AWS_REGION=$AWS_REGION" | tee outputs/01_region_echo.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. (Opcional) Crear el clúster EKS

- Si **ya tienes** un clúster EKS, solo actualizarás kubeconfig y validarás conectividad.  
- Si **no tienes** clúster, lo crearás con `eksctl` (Managed Node Group) y validarás que esté listo.

> **IMPORTANTE:** Crear un clúster genera costos (EKS + EC2). Elimínalo al final si es solo laboratorio.
{: .lab-note .important .compact}

#### Tarea 2.1 — Reutilizar clúster existente (kubeconfig + conectividad)

- {% include step_label.html %} Intenta apuntar `kubectl` a un clúster existente y valida conectividad.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION" | tee outputs/02_update_kubeconfig.txt
  ```
  ```bash
  kubectl config current-context | tee outputs/02_kube_context.txt
  ```
  ```bash
  kubectl get nodes -o wide | tee outputs/02_nodes_existing.txt
  ```

- {% include step_label.html %} Si `kubectl get nodes` funcionó, continúa con la **Tarea 3**. Si falló, crea el clúster en la **Tarea 2.2**.

  > **NOTA:** Un fallo típico aquí es `ResourceNotFoundException` (cluster no existe) o credenciales/region incorrectas.
  {: .lab-note .info .compact}

#### Tarea 2.2 — Crear clúster con eksctl (Managed Node Group)

- {% include step_label.html %} (Recomendado) Lista versiones soportadas en tu región antes de elegir `K8S_VERSION`.

  ```bash
  export AWS_REGION="us-west-2"
  ```
  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el clúster EKS (2 nodos administrados).

  ```bash
  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$K8S_VERSION" \
    --managed --nodegroup-name "mng-1" \
    --node-type "t3.medium" --nodes 2 --nodes-min 2 --nodes-max 3 | tee outputs/02_eksctl_create_cluster.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Configura kubeconfig y valida el estado del clúster.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION" | tee outputs/02_update_kubeconfig_after_create.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl cluster-info | tee outputs/02_cluster_info.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get nodes -o wide | tee outputs/02_nodes_after_create.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get ns | tee outputs/02_namespaces_after_create.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Valida desde AWS el estado del clúster (`ACTIVE`).

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.status" --output text | tee outputs/02_cluster_status.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica si tu identidad puede usar **impersonation** (para `--as`). Debe responder `yes` idealmente.

  ```bash
  kubectl auth can-i impersonate serviceaccounts | tee outputs/01_can_i_impersonate_sa.txt
  ```
  ```bash
  kubectl auth can-i impersonate users | tee outputs/01_can_i_impersonate_users.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Namespaces + app de prueba (Deployment/Service) + objetos para pruebas (ConfigMaps/Secrets)

Crearás dos namespaces (**dev** y **prod**) y desplegarás una app mínima en ambos. También crearás **ConfigMaps** y **Secrets** para probar *least privilege* y `resourceNames`.

> **NOTA (CKAD):** Patrón: `apply` → `rollout status` → `get -o wide` → `describe` → `events`.
{: .lab-note .info .compact}

#### Tarea 3.1

- {% include step_label.html %} Crea los namespaces **dev** y **prod** (idempotente).

  ```bash
  kubectl create ns "$NS_DEV"  --dry-run=client -o yaml | kubectl apply -f -
  kubectl create ns "$NS_PROD" --dry-run=client -o yaml | kubectl apply -f -
  ```
  ```bash
  kubectl get ns | egrep "$NS_DEV|$NS_PROD" | tee outputs/03_namespaces_created.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea ConfigMaps y Secret en **dev** (para pruebas de `resourceNames` y “no secrets por defecto”).

  ```bash
  kubectl -n "$NS_DEV" create configmap app-config --from-literal=MESSAGE="hola-dev" --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_DEV" create configmap other-config --from-literal=FEATURE="on" --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_DEV" create secret generic db-secret --from-literal=password="super-secret" --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea ConfigMaps y Secret en **prod**.

  ```bash
  kubectl -n "$NS_PROD" create configmap app-config --from-literal=MESSAGE="hola-prod" --dry-run=client -o yaml | kubectl apply -f -
  ```
  ```bash
  kubectl -n "$NS_PROD" create configmap other-config --from-literal=FEATURE="off" --dry-run=client -o yaml | kubectl apply -f -
  ```
  ```bash
  kubectl -n "$NS_PROD" create secret generic db-secret --from-literal=password="prod-secret" --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el manifiesto de la app demo (Deployment + Service) en `k8s/app.yaml`.

  ```bash
  cat > k8s/app.yaml <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: web
    labels:
      app: web
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: web
    template:
      metadata:
        labels:
          app: web
      spec:
        containers:
        - name: nginx
          image: nginx:1.27
          ports:
          - containerPort: 80
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: web
    labels:
      app: web
  spec:
    selector:
      app: web
    ports:
    - name: http
      port: 80
      targetPort: 80
  EOF
  ```

- {% include step_label.html %} Aplica la app en **dev** y **prod** y espera rollouts.

  ```bash
  kubectl -n "$NS_DEV"  apply -f k8s/app.yaml | tee outputs/03_apply_app_dev.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_PROD" apply -f k8s/app.yaml | tee outputs/03_apply_app_prod.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_DEV"  rollout status deploy/web | tee outputs/03_rollout_dev.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_PROD" rollout status deploy/web | tee outputs/03_rollout_prod.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Evidencia final de inventario (dev y prod).

  ```bash
  kubectl -n "$NS_DEV"  get deploy,po,svc,cm,secret -o wide | tee outputs/03_dev_inventory.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_PROD" get deploy,po,svc,cm,secret -o wide | tee outputs/03_prod_inventory.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Crear ServiceAccounts (identidades) por “rol” (dev/prod)

Crearás identidades para simular equipos: CI, observabilidad, debug, config-updater y un viewer en prod.

#### Tarea 4.1

- {% include step_label.html %} Crea ServiceAccounts en **dev**.

  ```bash
  kubectl -n "$NS_DEV" create sa sa-ci --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_DEV" create sa sa-observer --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_DEV" create sa sa-debug --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_DEV" create sa sa-config --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea ServiceAccount en **prod** (solo lectura).

  ```bash
  kubectl -n "$NS_PROD" create sa sa-prod-view --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda evidencia (inventario de SAs).

  ```bash
  kubectl -n "$NS_DEV"  get sa | tee outputs/04_sa_dev.txt
  ```
  ```bash
  kubectl -n "$NS_PROD" get sa | tee outputs/04_sa_prod.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Implementar Roles y RoleBindings (least privilege + subrecursos)

Crearás Roles separados por riesgo (deploy, logs, exec) y RoleBindings que asignan permisos a cada ServiceAccount.

> **NOTA (CKAD):** Role vs RoleBinding (namespaced) + subrecursos (`pods/log`, `pods/exec`) son puntos típicos de examen.
{: .lab-note .info .compact}

#### Tarea 5.1 — RBAC en dev (Roles)

- {% include step_label.html %} Crea `rbac/rbac-dev.yaml` con 3 Roles: deployer, logs y exec.

  ```bash
  cat > rbac/rbac-dev.yaml <<'EOF'
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: dev-deployer
    namespace: rbac-dev
  rules:
  # Deployments/ReplicaSets: CI puede crear/actualizar/eliminar workloads
  - apiGroups: ["apps"]
    resources: ["deployments","replicasets"]
    verbs: ["get","list","watch","create","update","patch","delete"]
  # Services/ConfigMaps: CI puede gestionar config no sensible
  - apiGroups: [""]
    resources: ["services","configmaps"]
    verbs: ["get","list","watch","create","update","patch","delete"]
  # Pods/Events: solo lectura (para troubleshooting)
  - apiGroups: [""]
    resources: ["pods","events"]
    verbs: ["get","list","watch"]
  # Least privilege: NO secrets, NO RBAC objects, NO exec
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: dev-logs
    namespace: rbac-dev
  rules:
  - apiGroups: [""]
    resources: ["pods","pods/log","events"]
    verbs: ["get","list","watch"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: dev-exec
    namespace: rbac-dev
  rules:
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]
  EOF
  ```

- {% include step_label.html %} Aplica el RBAC en dev.

  ```bash
  kubectl apply -f rbac/rbac-dev.yaml | tee outputs/05_apply_rbac_dev.txt
  ```
  {% include step_image.html %}

#### Tarea 5.2 — RBAC en dev (RoleBindings)

- {% include step_label.html %} Crea `rbac/rbac-dev-bindings.yaml` (asignación por identidad).

  ```bash
  cat > rbac/rbac-dev-bindings.yaml <<'EOF'
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: rb-ci-deployer
    namespace: rbac-dev
  subjects:
  - kind: ServiceAccount
    name: sa-ci
    namespace: rbac-dev
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: dev-deployer
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: rb-observer-logs
    namespace: rbac-dev
  subjects:
  - kind: ServiceAccount
    name: sa-observer
    namespace: rbac-dev
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: dev-logs
  ---
  # Debug: logs + exec (separados a propósito)
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: rb-debug-logs
    namespace: rbac-dev
  subjects:
  - kind: ServiceAccount
    name: sa-debug
    namespace: rbac-dev
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: dev-logs
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: rb-debug-exec
    namespace: rbac-dev
  subjects:
  - kind: ServiceAccount
    name: sa-debug
    namespace: rbac-dev
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: dev-exec
  EOF
  ```

- {% include step_label.html %} Aplica los bindings en dev.

  ```bash
  kubectl apply -f rbac/rbac-dev-bindings.yaml | tee outputs/05_apply_rbac_dev_bindings.txt
  ```
  {% include step_image.html %}

#### Tarea 5.3 — RBAC en prod (viewer read-only)

- {% include step_label.html %} Crea `rbac/rbac-prod.yaml` (Role + RoleBinding).

  ```bash
  cat > rbac/rbac-prod.yaml <<'EOF'
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: prod-viewer
    namespace: rbac-prod
  rules:
  - apiGroups: ["apps"]
    resources: ["deployments","replicasets"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["pods","pods/log","services","configmaps","events"]
    verbs: ["get","list","watch"]
  # NO secrets, NO exec
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: rb-prod-view
    namespace: rbac-prod
  subjects:
  - kind: ServiceAccount
    name: sa-prod-view
    namespace: rbac-prod
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: prod-viewer
  EOF
  ```

- {% include step_label.html %} Aplica el RBAC en prod.

  ```bash
  kubectl apply -f rbac/rbac-prod.yaml | tee outputs/05_apply_rbac_prod.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Evidencia de inventario RBAC por namespace.

  ```bash
  kubectl -n "$NS_DEV"  get role,rolebinding | tee outputs/05_rbac_dev_inventory.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_PROD" get role,rolebinding | tee outputs/05_rbac_prod_inventory.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Validación con `kubectl auth can-i` (impersonation)

Usarás `kubectl auth can-i` con `--as` como “unit tests” de permisos. Guardarás evidencia (yes/no).

> **IMPORTANTE:** Si `--as` falla por falta de permiso **impersonate**, revisa la salida de la Tarea 1 (`outputs/01_can_i_impersonate_*.txt`).
{: .lab-note .important .compact}

#### Tarea 6.1

- {% include step_label.html %} Ejecuta pruebas **positivas/negativas** y guarda evidencia.

  - **CI en dev: puede modificar deployments/configmaps, pero NO secrets**

  ```bash
  kubectl auth can-i patch deployments -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-ci | tee outputs/06_ci_patch_deploy.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i patch configmaps -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-ci | tee outputs/06_ci_patch_configmaps.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i get secrets -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-ci | tee outputs/06_ci_get_secrets.txt
  ```
  {% include step_image.html %}

  - **Observer en dev: puede ver logs, NO puede borrar deployments**
  
  ```bash
  kubectl auth can-i get pods/log -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-observer | tee outputs/06_observer_logs.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i delete deployments -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-observer | tee outputs/06_observer_delete_deploy.txt
  ```
  {% include step_image.html %}

  - **Prod viewer: solo lectura (NO delete)**

  ```bash
  kubectl auth can-i delete pods -n "$NS_PROD" --as=system:serviceaccount:"$NS_PROD":sa-prod-view | tee outputs/06_prod_delete_pods.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i get pods/log -n "$NS_PROD" --as=system:serviceaccount:"$NS_PROD":sa-prod-view | tee outputs/06_prod_logs.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. RBAC avanzado: `resourceNames` + pruebas reales (Allowed vs Forbidden) + separación logs/exec

Aplicarás controles finos:
- 1) restringir updates a **un ConfigMap específico** con `resourceNames`  
- 2) demostrar “permitido vs denegado” con `kubectl patch`  
- 3) reforzar subrecursos como permisos separados (`pods/log` vs `pods/exec`) con comandos reales

> **NOTA (CKAD):** Practicarás fallas **Forbidden** intencionales y aprenderás a justificar "qué regla falta o sobra".
{: .lab-note .info .compact}

#### Tarea 7.1 — `resourceNames` para proteger objetos por nombre

- {% include step_label.html %} Crea Role + RoleBinding para que `sa-config` solo pueda **patch/update** `app-config` (pero no `other-config`).

  ```bash
  cat > rbac/rbac-config-restrict.yaml <<'EOF'
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: dev-config-updater
    namespace: rbac-dev
  rules:
  # Regla 1: leer/listar configmaps (sin resourceNames)
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get","list","watch"]
  # Regla 2: modificar SOLO app-config (con resourceNames)
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]
    verbs: ["update","patch"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: rb-config-updater
    namespace: rbac-dev
  subjects:
  - kind: ServiceAccount
    name: sa-config
    namespace: rbac-dev
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: dev-config-updater
  EOF
  ```

- {% include step_label.html %} Aplica el RBAC de `resourceNames`.

  ```bash
  kubectl apply -f rbac/rbac-config-restrict.yaml | tee outputs/07_apply_rbac_resourceNames.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba positiva: `sa-config` **sí** puede parchear `app-config`.

  ```bash
  kubectl -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-config patch configmap app-config -p '{"data":{"MESSAGE":"hola-dev-actualizado"}}' | tee outputs/07_patch_app_config_ok.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba negativa: `sa-config` **NO** puede parchear `other-config` (Forbidden).

  ```bash
  kubectl -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-config patch configmap other-config -p '{"data":{"FEATURE":"maybe"}}' 2>&1 | tee outputs/07_patch_other_config_forbidden.txt
  ```
  {% include step_image.html %}

#### Tarea 7.2 — Subrecursos: logs vs exec (control de acciones sensibles)

- {% include step_label.html %} Obtén un Pod `web` en dev (sin inventar el nombre).

  ```bash
  POD_WEB="$(kubectl -n "$NS_DEV" get pod -l app=web -o jsonpath='{.items[0].metadata.name}')"
  echo "POD_WEB=$POD_WEB" | tee outputs/07_pod_web_name.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_DEV" get pod "$POD_WEB" -o wide | tee outputs/07_pod_web_wide.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} `sa-observer` puede **logs**, pero **no** puede exec.

  ```bash
  kubectl -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-observer logs pod/"$POD_WEB" --tail=5 | tee outputs/07_observer_logs_ok.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-observer exec pod/"$POD_WEB" -- sh -c "id" 2>&1 | tee outputs/07_observer_exec_forbidden.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} `sa-debug` **sí** puede realizar exec.

  ```bash
  kubectl -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-debug exec pod/"$POD_WEB" -- sh -c "echo OK && id" | tee outputs/07_debug_exec_ok.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Evidencia final de **can-i** para `resourceNames` y subrecursos (unit tests finales).

  ```bash
  kubectl auth can-i patch configmaps -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-config | tee outputs/07_can_i_patch_configmaps_sa_config.txt
  ```
  ```bash
  kubectl auth can-i patch configmap/app-config -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-config | tee outputs/07_can_i_patch_app_config_sa_config.txt
  ```
  ```bash
  kubectl auth can-i patch configmap/other-config -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-config | tee outputs/07_can_i_patch_other_config_sa_config.txt
  ```
  ```bash
  kubectl auth can-i get pods/log -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-observer | tee outputs/07_can_i_logs_sa_observer.txt
  ```
  ```bash
  kubectl auth can-i create pods/exec -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-observer | tee outputs/07_can_i_exec_sa_observer.txt
  ```
  ```bash
  kubectl auth can-i create pods/exec -n "$NS_DEV" --as=system:serviceaccount:"$NS_DEV":sa-debug | tee outputs/07_can_i_exec_sa_debug.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Troubleshooting final

Consolidarás un checklist para diagnosticar errores **Forbidden**.

#### Tarea 8.1 — Checklist de diagnóstico (cuando sale Forbidden)

- {% include step_label.html %} Inventario rápido (qué existe y dónde).

  ```bash
  kubectl -n "$NS_DEV"  get sa,role,rolebinding | tee outputs/08_dev_rbac_inventory.txt
  ```
  ```bash
  kubectl -n "$NS_PROD" get sa,role,rolebinding | tee outputs/08_prod_rbac_inventory.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inspecciona bindings (subject + roleRef).

  ```bash
  kubectl -n "$NS_DEV"  describe rolebinding rb-ci-deployer | tee outputs/08_describe_rb_ci.txt
  ```
  ```bash
  kubectl -n "$NS_DEV"  describe rolebinding rb-debug-exec | tee outputs/08_describe_rb_debug_exec.txt
  ```
  ```bash
  kubectl -n "$NS_PROD" describe rolebinding rb-prod-view | tee outputs/08_describe_rb_prod_view.txt
  ```
  ```bash
  kubectl -n "$NS_DEV"  describe rolebinding rb-config-updater 2>/dev/null | tee outputs/08_describe_rb_config_updater.txt || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa reglas exactas del Role (qué permisos están definidos).

  ```bash
  kubectl -n "$NS_DEV" get role dev-deployer -o yaml | tee outputs/08_role_dev_deployer.yaml
  ```
  ```bash
  kubectl -n "$NS_DEV" get role dev-logs -o yaml | tee outputs/08_role_dev_logs.yaml
  ```
  ```bash
  kubectl -n "$NS_DEV" get role dev-exec -o yaml | tee outputs/08_role_dev_exec.yaml
  ```
  ```bash
  kubectl -n "$NS_PROD" get role prod-viewer -o yaml | tee outputs/08_role_prod_viewer.yaml
  ```

- {% include step_label.html %} (Recomendado) Eventos recientes por namespace (pueden explicar problemas colaterales).

  ```bash
  kubectl -n "$NS_DEV"  get events --sort-by=.lastTimestamp | tail -n 30 | tee outputs/08_events_dev_tail.txt
  ```
  ```bash
  kubectl -n "$NS_PROD" get events --sort-by=.lastTimestamp | tail -n 30 | tee outputs/08_events_prod_tail.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9. Limpieza del laboratorio

Limpiarás recursos (namespaces y, si aplica, clúster).

#### Tarea 9.1

- {% include step_label.html %} Elimina los namespaces del laboratorio (remueve todo lo namespaced: app, SAs, Roles, RoleBindings, CM/Secrets).

  ```bash
  kubectl delete ns "$NS_DEV" "$NS_PROD" --ignore-not-found | tee outputs/08_delete_namespaces.txt
  ```

- {% include step_label.html %} Verifica que ya no existan.

  ```bash
  kubectl get ns | egrep "$NS_DEV|$NS_PROD" || echo "OK: namespaces eliminados" | tee outputs/08_verify_namespaces_deleted.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el clúster creado por `eksctl`.

  > **NOTA:** El cluster tardara **9 minutos** aproximadamente en eliminarse.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el cluster se haya eliminado correctamente.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || echo "Cluster eliminado"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}