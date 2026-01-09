---
layout: lab
title: "Práctica 3: Identificación y corrección de anti-patrones en un clúster EKS"
permalink: /lab3/lab3/
images_base: /labs/lab3/img
duration: "60 minutos"
objective:
  - "Desplegar intencionalmente una aplicación con anti-patrones comunes en Kubernetes/EKS (probes incorrectas, :latest, sin requests/limits, RBAC excesivo, sin PDB, sin hardening de securityContext, tokens montados innecesariamente) y luego diagnosticarlos con kubectl (events/logs/describe) para aplicar correcciones alineadas a buenas prácticas y al enfoque CKAD (workloads, configuración declarativa, troubleshooting y seguridad básica)."
prerequisites:
  - "Amazon EKS accesible con kubectl"
  - "Windows + Visual Studio Code + Terminal GitBash."
  - "Herramientas: AWS CLI v2, kubectl, eksctl."
  - "Docker Desktop"
  - "Permisos Kubernetes: crear namespaces, deployments, services, RBAC, PDB"
  - "Permisos AWS"
introduction:
  - "En Kubernetes, muchos problemas que parecen “fallas del clúster” suelen ser anti-patrones en manifiestos: liveness mal configurada, pods BestEffort por falta de requests/limits, permisos RBAC demasiado amplios, o imágenes sin versionar. En esta práctica provocarás síntomas reales y los corregirás con cambios declarativos y troubleshooting con kubectl."
slug: lab3
lab_number: 3
final_result: >
  Al finalizar habrás desplegado una app “mala” con anti-patrones reales, diagnosticado sus síntomas con kubectl y aplicado correcciones de confiabilidad y seguridad (probes correctas, recursos, imagen versionada, PDB, RBAC least privilege, token de SA controlado, securityContext endurecido, dejando un patrón reproducible para revisar manifiestos antes de producción.
notes:
  - "Esta práctica es de laboratorio: incluye RBAC excesivo y configuraciones inseguras de forma intencional. No las uses en producción."
  - "Para evitar fricción/costos, usaremos imágenes públicas (no es necesario ECR)."
  - "NetworkPolicy solo funciona si tu clúster tiene un plugin/CNI que la implemente (en EKS depende de la configuración). Por eso es opcional y la validación lo indica."
  - "CKAD: enfoque 100% developer-side (Deployment/Service/ConfigMap, probes, recursos, SA/RBAC, troubleshooting)."
references:
  - text: "Kubernetes - Probes (liveness/readiness/startup)"
    url: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
  - text: "Kubernetes - Images / imagePullPolicy / :latest"
    url: https://kubernetes.io/docs/concepts/containers/images/
  - text: "Kubernetes - Resource requests/limits"
    url: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
  - text: "Kubernetes - QoS Classes"
    url: https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/
  - text: "Kubernetes - PodDisruptionBudget"
    url: https://kubernetes.io/docs/tasks/run-application/configure-pdb/
  - text: "Kubernetes - RBAC (Using RBAC Authorization)"
    url: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
  - text: "Kubernetes - ServiceAccounts (automountServiceAccountToken)"
    url: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
  - text: "Kubernetes - Pod Security Admission / Pod Security Standards"
    url: https://kubernetes.io/docs/concepts/security/pod-security-admission/
  - text: "Kubernetes - NetworkPolicy"
    url: https://kubernetes.io/docs/concepts/services-networking/network-policies/
  - text: "AWS - Amazon EKS Best Practices Guides (Security)"
    url: https://aws.github.io/aws-eks-best-practices/
prev: /lab2/lab2/
next: /lab4/lab4/
---

---

### Tarea 1. Preparar la carpeta de la práctica

Confirmarás identidad/cuenta/región, asegurarás que `kubectl` apunta al clúster correcto.

> **IMPORTANTE:** Esta práctica crea anti-patrones a propósito (incluye RBAC muy peligroso). Aplícalo solo en un clúster de laboratorio.
{: .lab-note .important .compact}

> **NOTA (CKAD):** Practicarás troubleshooting con `kubectl describe`, `kubectl get events`, `kubectl logs`, JSONPath, y correcciones declarativas en manifiestos (Deployment/Service/PDB/RBAC).
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo asignado al curso con el usuario que tenga permisos administrativos. 

- {% include step_label.html %} Abre el **`Visual Studio Code`**. Lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- {% include step_label.html %} Una vez abierto **VSCode**, da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  {% include step_image.html %}

- {% include step_label.html %} Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar dentro de la carpeta del curso llamada **labs-eks-ckad** en la terminal de **VSCode**.

  > **Nota.** Si te quedaste en el directorio de una práctica, usa **`cd ..`** para retornar a la raíz de los laboratorios.
  {: .lab-note .info .compact}

- {% include step_label.html %} Ahora crea el directorio para trabajar en la Práctica 2.

  ```bash
  mkdir lab03 && cd lab03
  ```

- {% include step_label.html %} Valida en el **Explorador** de archivos dentro de **VSCode** que se haya creado el directorio.

  {% include step_image.html %}

- {% include step_label.html %} Ejecuta el siguiente comando para crear el resto de los archivos y directorios de la practica.

  > **NOTA:** El comando se ejecuta desde el directorio raíz **lab03**
  {: .lab-note .info .compact}

  ```bash
  mkdir -p k8s scripts outputs
  mkdir -p k8s/bad k8s/fixed
  ```

- {% include step_label.html %} Confirma la estructura de los directorios creados, ejecuta el siguiente comando

  > **Nota.** Organizar cada práctica en carpetas separadas facilita la gestión de ejemplos y evita confusiones.
  {: .lab-note .info .compact}

  ```bash
  find . -maxdepth 3 -type d | sort
  ```

  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear un clúster EKS para la práctica

Crearás un clúster EKS con Managed Node Group usando `eksctl`.

> **IMPORTANTE:** Si ya tienes un clúster EKS y `kubectl`, puedes conectarte. **Omite** esta tarea si es necesario.
{: .lab-note .important .compact}

#### Tara 2.1

- {% include step_label.html %} (Recomendado) Revisa las versiones soportadas en la región antes de elegir `K8S_VERSION`.

  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```

- {% include step_label.html %} Asegúrate de que la versión elegida aparece en la lista, usaremos la version **`1.33`**.

- {% include step_label.html %} Define las variables del clúster (región/nombres/versión) y tamaño mínimo de nodos.

  ```bash
  export AWS_REGION="us-west-2"
  export CLUSTER_NAME="eks-antipatterns-lab"
  export NODEGROUP_NAME="mng-1"
  export K8S_VERSION="1.33"
  ```

- {% include step_label.html %} Ejecuta el siguiente comando para validar el guardado correcto de las variables.

  ```bash
  echo "AWS_REGION=$AWS_REGION"
  echo "CLUSTER_NAME=$CLUSTER_NAME"
  echo "NODEGROUP_NAME=$NODEGROUP_NAME"
  echo "K8S_VERSION=$K8S_VERSION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica la identidad de AWS (reduce riesgo de crear recursos en la cuenta equivocada).

  ```bash
  aws sts get-caller-identity
  aws configure get region || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el `Account` y `Arn` sean los que se te asignaron al curso.

- {% include step_label.html %} Crea el clúster EKS con Managed Node Group (2 nodos) usando `eksctl`.

  > **NOTA:** `eksctl` crea VPC/subnets/SG/roles y el clúster con un Managed Node Group listo para workloads.
  {: .lab-note .info .compact}

  > **IMPORTANTE:** El cluster tardara aproximadamente **15 minutos** en crearse. Espera el proceso
  {: .lab-note .important .compact}

  ```bash
  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$K8S_VERSION" \
    --managed \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-type t3.medium \
    --nodes 2 --nodes-min 2 --nodes-max 3
  ```
  {% include step_image.html %}

- {% include step_label.html %} Configura kubeconfig y el contexto para entrar a kubernetes.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  kubectl config current-context
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida la conectividad a Kubernetes.

  ```bash
  kubectl cluster-info
  ```
  ```bash
  kubectl get nodes -o wide
  ```
  ```bash
  kubectl get ns
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el cluster este correctamente creado, con los siguientes comandos.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.status" --output text
  ```
  ```bash
  aws eks describe-nodegroup --cluster-name "$CLUSTER_NAME" --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION" --query "nodegroup.status" --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Recomendado) Guarda un snapshot inicial del estado del clúster

  ```bash
  kubectl get nodes -o wide | tee outputs/baseline_nodes.txt
  ```
  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 10 | tee outputs/baseline_events_tail.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Preparación del entorno y baseline

Validarás identidad AWS/cluster context, crearás variables y tomarás un baseline para comparar antes/después (nodos, eventos, namespaces).

#### Tarea 3.1

- {% include step_label.html %} Apunta el `kubectl` al clúster EKS correcto. Si ya actualizaste el contexto puedes omitir este paso.

  > **NOTA:** Evita aplicar RBAC/PSA/Policies en el clúster equivocado.
  {: .lab-note .info .compact}

  ```bash
  export CLUSTER_NAME="${CLUSTER_NAME:-TU-CLUSTER}"
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  ```bash
  kubectl get nodes -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Genera un snapshot rápido (ruido previo(antipatrones) vs lo que tú generes).

  ```bash
  kubectl get ns | tee outputs/namespaces_before.txt
  ```
  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 15 | tee outputs/events_before_tail.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que puedes aplicar los manifiestos (dry-run).

  ```bash
  kubectl create ns anti --dry-run=client -o yaml | tee outputs/ns_anti_dryrun.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que no marque errores la validación

  ```bash
  kubectl apply --dry-run=client -f outputs/ns_anti_dryrun.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Desplegar una app con anti-patrones intencionales

Aplicarás manifiestos **malos** que provocan síntomas reales: reinicios por liveness mal definida, imagen `:latest`, sin requests/limits (QoS BestEffort) y RBAC peligrosamente amplio.

> **IMPORTANTE:** Esto es laboratorio. El ClusterRoleBinding a `cluster-admin` es un anti-patrón severo.
{: .lab-note .important .compact}

#### Tarea 4.1 

- {% include step_label.html %} Crea el namespace de laboratorio `anti`.

  ```bash
  cat > k8s/bad/00-namespace.yaml <<'EOF'
  apiVersion: v1
  kind: Namespace
  metadata:
    name: anti
  EOF

  kubectl apply -f k8s/bad/00-namespace.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida la configuración del namespace.

  ```bash
  kubectl get ns anti
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el anti-patrón RBAC: ServiceAccount con ClusterRoleBinding excesivo (`cluster-admin`).

  ```bash
  cat > k8s/bad/01-rbac-too-wide.yaml <<'EOF'
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: app-sa
    namespace: anti
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: anti-app-cluster-admin
  subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: anti
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: cluster-admin
  EOF

  kubectl apply -f k8s/bad/01-rbac-too-wide.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Realiza la validación de la configuración

  ```bash
  kubectl get clusterrolebinding anti-app-cluster-admin -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea los anti-patrones de confiabilidad: `:latest`, sin requests/limits, liveness mal configurada.

  > **NOTA:**
  - `nginx:latest`: no reproducible (puede cambiar sin aviso).
  - sin resources: QoS tiende a BestEffort.
  - liveness a `/healthz`: NGINX no la sirve por defecto → fallas → reinicios.
  {: .lab-note .info .compact}

  ```bash
  cat > k8s/bad/02-deploy-bad.yaml <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: web
    namespace: anti
    labels:
      app: web
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: web
    template:
      metadata:
        labels:
          app: web
      spec:
        serviceAccountName: app-sa
        containers:
        - name: web
          image: nginx:latest
          ports:
          - containerPort: 80
          # Anti-patrón: liveness probe mal definida (ruta inexistente) → reinicios
          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: web
    namespace: anti
  spec:
    selector:
      app: web
    ports:
    - port: 80
      targetPort: 80
  EOF
  ```
  ```bash
  kubectl apply -f k8s/bad/02-deploy-bad.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa los síntomas (pods reiniciando / eventos).

  > **NOTA:** Deja ejecutando el segundo comando 1 minutos y detectaras los reinicios, inclusive podrias llegar a detectar **CrashLoopBAckOff** puedes romper el proceso con **`CTRL+C`**
  {: .lab-note .info .compact}

  ```bash
  kubectl get all -n anti
  ```
  ```bash
  kubectl get pods -n anti -w
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida los primeros indicios de los eventos.

  ```bash
  kubectl get events -n anti --sort-by=.lastTimestamp | tail -n 25
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Diagnóstico guiado: síntomas → causa

Usarás `kubectl describe`, `events` y JSONPath para mapear síntomas a causas: liveness incorrecta, uso de `:latest`, falta de requests/limits (QoS) y RBAC demasiado amplio.

#### Tarea 5.1

- {% include step_label.html %} Identifica el pod y revisa los detalles con `describe` (casi siempre muestra la razón).

  ```bash
  POD="$(kubectl get pod -n anti -l app=web -o jsonpath='{.items[0].metadata.name}')"
  ```
  ```bash
  echo "POD=$POD"
  ```
  ```bash
  kubectl describe pod "$POD" -n anti | sed -n '1,240p' | tee outputs/describe_pod_bad.txt
  ```
  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} Confirma que la liveness está fallando (restarts + eventos).

  ```bash
  echo -n "Restarts: "
  kubectl get pod "$POD" -n anti -o jsonpath='{.status.containerStatuses[0].restartCount}'; echo
  ```
  {% include step_image.html %}
  ```bash
  kubectl get events -n anti --sort-by=.lastTimestamp | tail -n 30 | tee outputs/events_anti_tail.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Detecta el anti-patrón `:latest` y `imagePullPolicy`.

  ```bash
  echo -n "Image: "
  kubectl get deploy web -n anti -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
  ```
  {% include step_image.html %}
  ```bash
  echo -n "imagePullPolicy: "
  kubectl get deploy web -n anti -o jsonpath='{.spec.template.spec.containers[0].imagePullPolicy}'; echo
  ```
  {% include step_image.html %}

- {% include step_label.html %} Detecta la falta de requests/limits → QoS.

  ```bash
  echo -n "QoS: "
  kubectl get pod "$POD" -n anti -o jsonpath='{.status.qosClass}'; echo
  ```
  ```bash
  echo -n "Resources: "
  kubectl get deploy web -n anti -o jsonpath='{.spec.template.spec.containers[0].resources}'; echo
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma el RBAC excesivo y prueba los permisos con `kubectl auth can-i`.

  ```bash
  kubectl describe clusterrolebinding anti-app-cluster-admin | sed -n '1,160p' | tee outputs/describe_crb_bad.txt
  ```
  ```bash
  kubectl auth can-i --as=system:serviceaccount:anti:app-sa get pods -A
  ```
  ```bash
  kubectl auth can-i --as=system:serviceaccount:anti:app-sa delete ns
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ejecuta los siguientes comandos y guarda la evidencia.

  ```bash
  {
    echo "Image:"; kubectl get deploy web -n anti -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
    echo "QoS:";   kubectl get pod "$POD" -n anti -o jsonpath='{.status.qosClass}'; echo
    echo "Restarts:"; kubectl get pod "$POD" -n anti -o jsonpath='{.status.containerStatuses[0].restartCount}'; echo
  } | tee outputs/diagnostic_checklist.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Correcciones de confiabilidad: probes, recursos, imagen versionada y PDB

Estabilizarás el workload con cambios declarativos: imagen versionada (sin `latest`), probes correctas, requests/limits y PDB para disponibilidad durante disrupciones.

> **NOTA:** Para que `runAsNonRoot` funcione sin trucos, usaremos `nginxinc/nginx-unprivileged` (escucha en 8080).
{: .lab-note .info .compact}

#### Tarea 6.1

- {% include step_label.html %} Crea el archivo Deployment/Service “fixed” (imagen versionada + recursos + probes correctas).

  > **NOTA:** El comando se ejecuta desde el directorio raíz **lab03**
  {: .lab-note .info .compact}
  
  ```bash
  cat > k8s/fixed/02-deploy-fixed.yaml <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: web
    namespace: anti
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
        serviceAccountName: app-sa
        containers:
        - name: web
          image: nginxinc/nginx-unprivileged:1.27.4-alpine
          ports:
          - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: web
    namespace: anti
  spec:
    selector:
      app: web
    ports:
    - port: 80
      targetPort: 8080
  EOF
  ```

- {% include step_label.html %} Realiza la validación con el comando **--dry-run**

  ```bash
  kubectl apply --dry-run=client -f k8s/fixed/02-deploy-fixed.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el archivo con el objeto **PodDisruptionBudget (PDB).**

  > **NOTA:** El comando se ejecuta desde el directorio raíz **lab03**
  {: .lab-note .info .compact}

  ```bash
  cat > k8s/fixed/03-pdb.yaml <<'EOF'
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata:
    name: web-pdb
    namespace: anti
  spec:
    minAvailable: 1
    selector:
      matchLabels:
        app: web
  EOF
  ```
  {% include step_image.html %}

- {% include step_label.html %} Realiza la validación con el comando **--dry-run**

  ```bash
  kubectl apply --dry-run=client -f k8s/fixed/03-pdb.yaml
  ```
  ```bash
  kubectl explain pdb.spec.minAvailable
  ```
  {% include step_image.html %}

- {% include step_label.html %} Aplica las correcciones. Ejecuta los 2 comandos siguientes.

  ```bash
  kubectl apply -f k8s/fixed/02-deploy-fixed.yaml
  ```
  ```bash
  kubectl apply -f k8s/fixed/03-pdb.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera el rollout y verifica estabilidad (sin reinicios).

  ```bash
  kubectl rollout status deploy/web -n anti
  kubectl get pods -n anti -l app=web -o wide
  ```
  {% include step_image.html %}

 - {% include step_label.html %} Ahora valida que QoS ya no sea BestEffort.

    ```bash
    kubectl get pod -n anti -l app=web -o jsonpath='{.items[0].status.qosClass}'; echo
    ```
    {% include step_image.html %}

- {% include step_label.html %} Prueba el ednpoint sin el uso de LB, directamente con port-forward.

  ```bash
  kubectl -n anti port-forward svc/web 8083:80
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre otra terminal y realiza la prueba al **localhost:8080**.

  ```bash
  cd lab03
  curl -sS http://localhost:8083 | head -n 10
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Correcciones de seguridad: RBAC least privilege, token SA, securityContext, PSA.

Eliminarás el RBAC excesivo, aplicarás RBAC mínimo, deshabilitarás el automount del token si no se requiere, endurecerás `securityContext`, habilitarás etiquetas de PSA (enforce/warn) y, opcionalmente, aplicarás NetworkPolicy si está soportada.

#### Tarea 7.1

- {% include step_label.html %} Elimina el ClusterRoleBinding peligroso creado previamente.

  ```bash
  kubectl delete clusterrolebinding anti-app-cluster-admin
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que en efecto se haya eliminado.

  ```bash
  kubectl get clusterrolebinding | grep anti-app-cluster-admin || echo "OK: cluster-admin binding removido"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Aplica el ServiceAccount con `automountServiceAccountToken: false` y un RBAC mínimo.

  > **NOTA:** El comando se ejecuta desde el directorio raíz **lab03**
  {: .lab-note .info .compact}

  ```bash
  cat > k8s/fixed/01-rbac-least-priv.yaml <<'EOF'
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: app-sa
    namespace: anti
  automountServiceAccountToken: false
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: app-read-config
    namespace: anti
  rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get","list"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: app-read-config-binding
    namespace: anti
  subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: anti
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: app-read-config
  EOF
  ```
  ```bash
  kubectl apply -f k8s/fixed/01-rbac-least-priv.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que este correctamente configurado

  > **NOTA:** El resultado **false** es correcto ya que por defecto **NO** se montará el token del ServiceAccount en los Pods que usen app-sa (salvo casos especiales o si el Pod lo sobreescribe). Perfecto para “least privilege” y hardening.
  {: .lab-note .info .compact}

  ```bash
  kubectl get sa app-sa -n anti -o jsonpath='{.automountServiceAccountToken}'; echo
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora realiza una prueba de permisos para ver el funcionamiento de la buena practica.

  > **NOTA:**
  - Te dio **yes** porque tu Role permite get y list sobre configmaps en el namespace anti.
  - Debería salir **no**, porque tu RBAC es solo un Role en anti (no un ClusterRole) y no se le dio permisos sobre namespaces.
  {: .lab-note .info .compact}

  ```bash
  kubectl auth can-i --as=system:serviceaccount:anti:app-sa list configmaps -n anti
  ```
  ```bash
  kubectl auth can-i --as=system:serviceaccount:anti:app-sa delete namespace/anti
  ```
  {% include step_image.html %}

- {% include step_label.html %} Endurece el `securityContext` (seccomp, no privilege escalation, drop caps, runAsNonRoot).

  > **NOTA:** El comando se ejecuta desde el directorio raíz **lab03**
  {: .lab-note .info .compact}

  ```bash
  cat > k8s/fixed/04-securitycontext-mergepatch.yaml <<'EOF'
  spec:
    template:
      spec:
        securityContext:
          seccompProfile:
            type: RuntimeDefault
        containers:
        - name: web
          securityContext:
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
  EOF
  ```
  ```bash
  kubectl -n anti patch deployment web --type strategic --patch-file k8s/fixed/04-securitycontext-mergepatch.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Realiza la validacion con los siguientes comandos.

  ```bash
  kubectl -n anti get deploy web -o jsonpath='{.spec.template.spec.containers[0].securityContext.runAsNonRoot}'; echo
  kubectl -n anti rollout status deploy/web
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional recomendado) Etiqueta el namespace para PSA (enforce baseline + warn restricted).

  ```bash
  kubectl label ns anti pod-security.kubernetes.io/enforce=baseline --overwrite
  ```
  ```bash
  kubectl label ns anti pod-security.kubernetes.io/warn=restricted --overwrite
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa las etiquetas aplicadas al **ns**

  ```bash
  kubectl get ns anti --show-labels
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Validación final integral

Verificarás estabilidad, salud y ausencia de eventos críticos en el namespace `anti` después de las correcciones.

#### Tarea 8.1

- {% include step_label.html %} Verifica el rollout, pods, PDB y los eventos.

  ```bash
  kubectl rollout status deploy/web -n anti
  kubectl get pods -n anti -l app=web
  ```
  ```bash
  kubectl get pdb -n anti
  ```
  ```bash
  kubectl get events -n anti --sort-by=.lastTimestamp | tail -n 25
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que ya **NO** existe RBAC peligroso y que el SA está endurecido.

  ```bash
  kubectl get clusterrolebinding | grep anti-app-cluster-admin || echo "OK: sin cluster-admin binding"
  ```
  ```bash
  kubectl get sa app-sa -n anti -o jsonpath='{.automountServiceAccountToken}'; echo
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9. Limpieza del laboratorio

Eliminarás el namespace `anti` y, opcionalmente, el clúster si lo creaste solo para laboratorio.

#### Tarea 9.1

- {% include step_label.html %} Elimina el namespace `anti`.

  ```bash
  kubectl delete ns anti
  ```

- {% include step_label.html %} Verifica que realmente se haya eliminado el **namespace**

  ```bash
  kubectl get ns | grep anti || echo "OK: namespace anti eliminado"
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Si creaste el clúster SOLO para esta práctica, elimínalo con eksctl.

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