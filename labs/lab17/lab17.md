---
layout: lab
title: "Práctica 17: Hardening de Pods, Nodos y Secrets en EKS"
permalink: /lab17/lab17/
images_base: /labs/lab17/img
duration: "60 minutos"
objective: |
  Aplicar hardening práctico en Amazon EKS enfocándote en:
  - (1) Pods (Pod Security Standards/Admission + securityContext “restricted”).
  - (2) Nodos (reducir exposición a IMDS/instance profile y reforzar postura de host).
  - (3) Secrets (cifrado, consumo seguro y separación por namespace). Validarás con pruebas positivas/negativas y troubleshooting con comandos tipo CKAD (apply/get/describe/logs/exec/auth can-i/events).
prerequisites:
  - "Amazon EKS accesible con kubectl (contexto correcto) o eksctl para crear un clúster de laboratorio"
  - "Windows + Visual Studio Code + Terminal GitBash"
  - "Herramientas: AWS CLI v2, kubectl (eksctl opcional), curl"
  - "Permisos Kubernetes: crear Namespace, Deployment, Service, Secret, NetworkPolicy"
  - "(Opcional) Permisos AWS/EC2: describe-instances para validar IMDS"
cost:
  - "No hay costo directo por ‘hardening’. Pagas por EKS control plane y nodos (EC2/EBS) mientras existan."
  - "Si habilitas cifrado con KMS customer-managed, aplica pricing de AWS KMS."
introduction:
  "En Kubernetes, hardening = reducir superficie de ataque: impedir privilegios innecesarios en Pods (root, capabilities, privileged, hostNetwork/hostPath), evitar que Pods alcancen el Instance Metadata Service (IMDS) del nodo para robar credenciales del instance profile, y tratar Secrets como material sensible (cifrado, mínimo acceso, consumo seguro). En EKS, la combinación de Pod Security Admission (PSA) + manifests ‘restricted’ + postura de nodos/IMDS + buenas prácticas de Secrets reduce riesgos comunes en entornos multi‑tenant."
slug: lab17
lab_number: 17
final_result: |
  Al finalizar tendrás dos namespaces (seguro vs inseguro) y evidencia práctica de hardening:
  - (1) PSA bloquea Pods inseguros con errores claros,
  - (2) workloads cumplen “restricted” con securityContext (non-root, sin escalation, sin capabilities, seccomp RuntimeDefault),
  - (3) validas el riesgo/mitigación de IMDS (y cuándo NetworkPolicy sí/no aplica en tu CNI).
  - (4) Manejas Secrets con separación por namespace y consumo como volumen read-only,
  respaldado con pruebas positivas/negativas y comandos de diagnóstico estilo CKAD.
notes:
  - "CKAD: Muy útil para practicar troubleshooting (Forbidden/FailedCreate), lectura de eventos, y precisión en YAML."
  - "PSA/PSS: Si tu clúster es antiguo o PSA está deshabilitado, los ‘rechazos’ pueden no ocurrir. Aun así, los manifests ‘restricted’ siguen siendo una buena práctica."
  - "NetworkPolicy: su enforcement depende del motor (CNI/add-on). En EKS, revisa tu soporte de NetworkPolicy según la configuración del clúster."
references:
  - text: "Kubernetes - Pod Security Standards (baseline/restricted/privileged)"
    url: https://kubernetes.io/docs/concepts/security/pod-security-standards/
  - text: "Kubernetes - Pod Security Admission (labels enforce/warn/audit)"
    url: https://kubernetes.io/docs/concepts/security/pod-security-admission/
  - text: "Kubernetes - Security Context (runAsNonRoot, capabilities, seccomp, etc.)"
    url: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
  - text: "Kubernetes - Secrets (conceptos y consumo en Pods)"
    url: https://kubernetes.io/docs/concepts/configuration/secret/
  - text: "AWS - EKS Best Practices: Network security (NetworkPolicy engines y recomendaciones)"
    url: https://docs.aws.amazon.com/eks/latest/best-practices/network-security.html
  - text: "AWS - EKS: Network policies (cómo habilitar / troubleshooting)"
    url: https://docs.aws.amazon.com/eks/latest/userguide/network-policies.html
  - text: "AWS - EC2: Instance Metadata Service (IMDSv2, tokens, hop limit)"
    url: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html
  - text: "AWS - EKS: Encrypt Kubernetes secrets with a KMS key"
    url: https://docs.aws.amazon.com/eks/latest/userguide/encrypt-secrets.html
prev: /lab16/lab16/
next: /lab18/lab18/
---

---

## Convenciones del laboratorio

- **Todo** se ejecuta en **GitBash** dentro de **VS Code**.
- Estructura estándar por práctica:
  - `k8s/` manifiestos Kubernetes
  - `scripts/` comandos auxiliares (opcional)
  - `outputs/` evidencias (txt/json)
- Namespaces usados:
  - `hardening-insecure` (para observar riesgos).
  - `hardening-secure` (para aplicar guard rails + manifests restricted).

> **NOTA (CKAD):** En este lab practicarás el loop “aplico → falla → leo events/describe → ajusto YAML”, que es exactamente el patrón más útil cuando algo sale mal.
{: .lab-note .info .compact}

---

### Tarea 1. Preparar carpeta y validar herramientas

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo del curso con un usuario con permisos administrativos.

- {% include step_label.html %} Abre **Visual Studio Code** y la terminal integrada.

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Git Bash** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar en la carpeta raíz de tus laboratorios (por ejemplo **`labs-eks-ckad`**).

  > **NOTA:** Si vienes de otra práctica, usa `cd ..` hasta volver a la raíz.
  {: .lab-note .info .compact}

- {% include step_label.html %} Crea el directorio del laboratorio y la estructura estándar (idempotente).

  ```bash
  mkdir -p lab17 && cd lab17
  mkdir -p k8s scripts outputs
  mkdir -p k8s/secure k8s/insecure k8s/netpol
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma la estructura creada y guarda evidencia.

  ```bash
  find . -maxdepth 3 -type d | sort | tee outputs/00_dirs.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida herramientas y guarda evidencia (evita sorpresas en troubleshooting).

  ```bash
  aws --version 2>&1 | tee outputs/01_aws_version.txt || true
  ```
  ```bash
  kubectl version --client=true --output=yaml | tee outputs/01_kubectl_version.yaml
  ```
  ```bash
  eksctl version 2>&1 | tee outputs/01_eksctl_version.txt || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables base (ajusta `AWS_REGION` si está vacío).

  ```bash
  export AWS_REGION="us-west-2"
  export CLUSTER_NAME="eks-hardening-lab17"
  export K8S_VERSION="1.33"

  export NS_SECURE="hardening-secure"
  export NS_INSECURE="hardening-insecure"
  ```

- {% include step_label.html %} Valida variables críticas.

  ```bash
  echo "AWS_REGION=$AWS_REGION" | tee outputs/01_region.txt
  echo "CLUSTER_NAME=$CLUSTER_NAME"
  echo "NS_SECURE=$NS_SECURE NS_INSECURE=$NS_INSECURE"
  ```
  ```bash
  test -n "$AWS_REGION" && echo "OK: AWS_REGION=$AWS_REGION" || echo "ERROR: AWS_REGION vacío"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear un clúster EKS

> **IMPORTANTE:** Crear un clúster puede tardar varios minutos y genera costos (EKS + EC2/EBS). Si lo creas solo para laboratorio, elimínalo al final.
{: .lab-note .important .compact}

#### Tarea 2.1 - Reutilizar clúster existente: kubeconfig + conectividad

- {% include step_label.html %} Si **ya tienes** un clúster, apunta `kubectl` a tu clúster y valida conectividad.

> **Si `kubectl get nodes` funciona**, continúa con la Tarea 3. Si no, crea el clúster en la Tarea 2.2.
{: .lab-note .info .compact}

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION" 2>/dev/null || true
  ```
  ```bash
  kubectl config current-context | tee outputs/02_context.txt
  ```
  ```bash
  kubectl cluster-info | tee outputs/02_cluster_info.txt
  ```
  ```bash
  kubectl get nodes -o wide | tee outputs/02_nodes.txt
  ```
  {% include step_image.html %}

#### Tarea 2.2 - Crear clúster con eksctl: Managed Node Group

- {% include step_label.html %} Verifica versiones soportadas en tu región (elige `K8S_VERSION` real de tu región).

  ```bash
  export AWS_REGION="us-west-2"
  ```
  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el clúster EKS con 2 nodos administrados.

  ```bash
  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$K8S_VERSION" \
    --managed --nodegroup-name "mng-1" \
    --node-type "t3.medium" --nodes 2 --nodes-min 2 --nodes-max 3 | tee outputs/02_eksctl_create_cluster.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Configura `kubeconfig` y valida el estado del clúster.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  ```bash
  kubectl config current-context | tee outputs/02_context_after_create.txt
  ```
  ```bash
  kubectl cluster-info | tee outputs/02_cluster_info_after_create.txt
  ```
  ```bash
  kubectl get nodes -o wide | tee outputs/02_nodes_after_create.txt
  ```
  ```bash
  kubectl get ns | tee outputs/02_namespaces_after_create.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Valida estado del clúster desde AWS y guarda evidencia.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.status" --output text | tee outputs/02_cluster_status.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Baseline de clúster + permisos mínimos

#### Tarea 3.1

- {% include step_label.html %} Captura baseline de eventos recientes (para comparar si algo se rompe después).

  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 20 | tee outputs/03_events_tail_baseline.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida permisos mínimos requeridos (si falla aquí, falla el resto).

  ```bash
  kubectl auth can-i create namespace | tee outputs/03_can_i_create_ns.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i create deployment -n default | tee outputs/03_can_i_create_deploy.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i create secret -n default | tee outputs/03_can_i_create_secret.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl auth can-i create networkpolicy -n default | tee outputs/03_can_i_create_netpol.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Crear namespaces del laboratorio

#### Tarea 4.1

- {% include step_label.html %} Crea los namespaces `hardening-insecure` y `hardening-secure` (idempotente).

  ```bash
  kubectl create ns "$NS_INSECURE" --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}
  ```bash
  kubectl create ns "$NS_SECURE"   --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que existen y guarda evidencia.

  ```bash
  kubectl get ns | egrep "hardening-(insecure|secure)" | tee outputs/04_namespaces_created.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Activar guard rails con Pod Security Admission (PSA) en el namespace seguro

#### Tarea 5.1

- {% include step_label.html %} Etiqueta `hardening-secure` con PSA **restricted** (enforce + warn + audit).

  ```bash
  kubectl label ns "$NS_SECURE" pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/warn=restricted pod-security.kubernetes.io/audit=restricted --overwrite
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma labels y guarda evidencia.

  ```bash
  kubectl get ns "$NS_SECURE" --show-labels | tee outputs/05_psa_labels.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica autorización RBAC para crear Pods (solo “can-i”; PSA se valida en la Tarea 6).

  ```bash
  kubectl auth can-i create pods -n "$NS_SECURE" | tee outputs/05_can_i_create_pods_secure.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Prueba negativa: un Pod inseguro debe ser rechazado por PSA

#### Tarea 6.1

- {% include step_label.html %} Crea el manifiesto del Pod inseguro (hostNetwork + privileged + cap).

  ```bash
  cat > k8s/secure/01_pod_insecure.yaml <<'EOF'
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-insecure
    namespace: hardening-secure
  spec:
    hostNetwork: true
    containers:
    - name: test
      image: busybox:1.36
      command: ["sh","-c","sleep 3600"]
      securityContext:
        privileged: true
        allowPrivilegeEscalation: true
        capabilities:
          add: ["NET_ADMIN"]
  EOF
  ```

- {% include step_label.html %} Intenta aplicarlo (espera **ERROR** por PSA).

  ```bash
  kubectl apply -f k8s/secure/01_pod_insecure.yaml 2>&1 | tee outputs/06_apply_pod_insecure.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que NO se creó el Pod y captura eventos como evidencia.

  ```bash
  kubectl -n "$NS_SECURE" get pods | tee outputs/06_secure_pods_after_fail.txt
  ```
  ```bash
  kubectl -n "$NS_SECURE" get events --sort-by=.lastTimestamp | tail -n 25 | tee outputs/06_psa_reject_events.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Desplegar una app “restricted” con securityContext

> **NOTA:** PSS “restricted” exige *non-root*, *no privilege escalation*, *seccomp RuntimeDefault* y *capabilities drop*. `readOnlyRootFilesystem` es recomendado (no siempre requerido). Aquí lo incluimos; si tu app crashea, verás cómo ajustarlo.
{: .lab-note .info .compact}

#### Tarea 7.1

- {% include step_label.html %} Crea el manifiesto “secure” (Deployment + Service).

  ```bash
  cat > k8s/secure/02_web_secure.yaml <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: web-secure
    namespace: hardening-secure
    labels:
      app: web-secure
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: web-secure
    template:
      metadata:
        labels:
          app: web-secure
      spec:
        automountServiceAccountToken: false
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          runAsGroup: 3000
          seccompProfile:
            type: RuntimeDefault
        containers:
        - name: app
          image: ghcr.io/stefanprodan/podinfo:6.9.4
          command: ["./podinfo"]
          args: ["--port=9898"]
          ports:
          - containerPort: 9898
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 300m
              memory: 256Mi
          volumeMounts:
          - name: tmp
            mountPath: /tmp
          - name: data
            mountPath: /data
        volumes:
        - name: tmp
          emptyDir: {}
        - name: data
          emptyDir: {}
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: web-secure
    namespace: hardening-secure
  spec:
    selector:
      app: web-secure
    ports:
    - name: http
      port: 80
      targetPort: 9898
  EOF
  ```

- {% include step_label.html %} Aplica y espera el rollout (evidencia).

  ```bash
  kubectl apply -f k8s/secure/02_web_secure.yaml | tee outputs/07_apply_web_secure.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_SECURE" rollout status deploy/web-secure | tee outputs/07_rollout_web_secure.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_SECURE" get pods -o wide | tee outputs/07_pods_web_secure.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba el Service desde un Pod temporal (sin LoadBalancer).

  > **NOTA:** La prueba negativa de PSA funcionando. Tu kubectl run curl... crea un Pod sin securityContext y en un namespace con PSA restricted (enforce), así que el Admission lo bloquea.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$NS_SECURE" run curl --image=curlimages/curl:8.10.1 --restart=Never -it --rm --command -- sh -lc "curl -sS http://web-secure/healthz && echo; curl -sS http://web-secure/readyz && echo"
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Si algo falla) Diagnostica rápido con describe + events.

  ```bash
  kubectl -n "$NS_SECURE" describe deploy web-secure | sed -n '1,220p' | tee outputs/07_describe_web_secure.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_SECURE" get events --sort-by=.lastTimestamp | tail -n 25 | tee outputs/07_events_web_secure.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Fix rápido si crashea por rootfs read-only) Parchea `readOnlyRootFilesystem=false` y reintenta rollout.

  ```bash
  kubectl -n "$NS_SECURE" patch deploy web-secure --type='json' -p='[
    {"op":"replace","path":"/spec/template/spec/containers/0/securityContext/readOnlyRootFilesystem","value":false}
  ]' | tee outputs/07_patch_rootfs_if_needed.txt
  ```
  ```bash
  kubectl -n "$NS_SECURE" rollout status deploy/web-secure | tee outputs/07_rollout_after_patch.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Prueba negativa: un Deployment “sin hardening” debe ser bloqueado por PSA

#### Tarea 8.1

- {% include step_label.html %} Crea el manifiesto "malo" (sin securityContext).

  ```bash
  cat > k8s/secure/03_web_bad.yaml <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: web-bad
    namespace: hardening-secure
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: web-bad
    template:
      metadata:
        labels:
          app: web-bad
      spec:
        containers:
        - name: app
          image: ghcr.io/stefanprodan/podinfo:6.9.4
          ports:
          - containerPort: 9898
  EOF
  ```

- {% include step_label.html %} Aplica y observa (espera que falle la creación de Pods).

  ```bash
  kubectl apply -f k8s/secure/03_web_bad.yaml | tee outputs/08_apply_web_bad.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_SECURE" get deploy,rs,pods -o wide | tee outputs/08_get_web_bad.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Diagnostica con `describe` + `events` para ver exactamente qué exige restricted.

  ```bash
  kubectl -n "$NS_SECURE" describe deploy web-bad | sed -n '1,260p' | tee outputs/08_describe_web_bad.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_SECURE" get events --sort-by=.lastTimestamp | tail -n 35 | tee outputs/08_events_web_bad.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9. Riesgo de nodos: probar acceso a IMDS desde un Pod

> **NOTA:** IMDSv2 puede responder `401 Unauthorized` si no envías token; eso **aún** indica conectividad. Lo importante aquí es: ¿llega o no llega al endpoint 169.254.169.254?
{: .lab-note .info .compact}

#### Tarea 9.1

- {% include step_label.html %} Ejecuta una prueba de conectividad a IMDS desde el namespace inseguro.

  ```bash
  kubectl -n "$NS_INSECURE" run imds-test --image=curlimages/curl:8.10.1 --restart=Never --command -- sh -lc "set -e; echo '== IMDS =='; curl -sS -m 2 -i http://169.254.169.254/latest/meta-data/ || true; sleep 5" | tee outputs/09_create_imds_test.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa logs y guarda evidencia.

  ```bash
  kubectl -n "$NS_INSECURE" logs pod/imds-test | tee outputs/09_imds_baseline.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_INSECURE" get pod imds-test -o wide | tee outputs/09_imds_pod_wide.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura `describe` por si necesitas ver red, node, y errores.

  ```bash
  kubectl -n "$NS_INSECURE" describe pod imds-test | sed -n '1,220p' | tee outputs/09_describe_imds_test.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 10. Validar configuración IMDS en EC2 y documentar mitigaciones

> **IMPORTANTE:** En producción, no “parchees” instancias a mano: aplica IMDSv2/hop-limit en el **Launch Template** del NodeGroup y rota nodos (infra inmutable).
{: .lab-note .important .compact}

#### Tarea 10.1

- {% include step_label.html %} Extrae `providerID` de nodos y guarda evidencia.

  ```bash
  kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{""}{.spec.providerID}{""}{end}' | tee outputs/10_nodes_providerid.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Construye lista de Instance IDs (desde providerID).

  ```bash
  INSTANCE_IDS="$(awk -F'/' '{print $NF}' outputs/10_nodes_providerid.txt | tr ' ' ' ')"
  ```
  ```bash
  echo "INSTANCE_IDS=$INSTANCE_IDS" | tee outputs/10_instance_ids.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Consulta `MetadataOptions` en EC2 y guarda evidencia.

  ```bash
  aws ec2 describe-instances --instance-ids $INSTANCE_IDS --query "Reservations[].Instances[].MetadataOptions.[InstanceId,HttpTokens,HttpPutResponseHopLimit]" --output table | tee outputs/10_imds_metadataoptions.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Interpreta resultados (qué buscar) y guarda una nota.

  ```bash
  echo "Recomendado: HttpTokens=required y HttpPutResponseHopLimit=1" | tee outputs/10_imds_recommendation.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[9] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 11. Bloqueo de IMDS por red con NetworkPolicy

> **NOTA:** Esto solo funcionará si tu clúster **aplica** NetworkPolicies (depende de CNI/add-ons). Si no hay enforcement, verás que la prueba IMDS sigue respondiendo.
{: .lab-note .info .compact}

> **NOTA (EKS + VPC CNI):** El enforcement nativo de NetworkPolicy del VPC CNI no aplica a Pods “standalone” (creados con `kubectl run` sin controller). Por eso en la Tarea 9 usamos un **Deployment**.
{: .lab-note .info .compact}

#### Tarea 11.1 — Habilitar enforcement de NetworkPolicy (Amazon VPC CNI)

- {% include step_label.html %} Verifica si `vpc-cni` está instalado como add-on administrado (si responde, es add-on).

  ```bash
  aws eks describe-addon --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" --addon-name vpc-cni --query "addon.addonVersion" --output text 2>&1 | tee outputs/11_vpc_cni_addon_version_or_error.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Habilita NetworkPolicy en el add-on (configuración `enableNetworkPolicy=true`).

  > **Si este comando falla** (por permisos o porque tu CNI no es add-on), la prueba puede seguir funcionando **si** tienes otro motor de NetworkPolicy (Calico/Cilium). Si no tienes motor, verás que IMDS sigue respondiendo.
  {: .lab-note .info .compact}

  ```bash
  aws eks update-addon --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" --addon-name vpc-cni --resolve-conflicts PRESERVE --configuration-values '{"enableNetworkPolicy":"true"}' 2>&1 | tee outputs/11_enable_netpol_update_addon.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera a que `aws-node` haga rollout y valida que apareció `aws-network-policy-agent`.

  ```bash
  kubectl -n kube-system rollout status ds/aws-node | tee outputs/11_rollout_aws_node_after_enable.txt
  ```
  ```bash
  kubectl -n kube-system get pods -l k8s-app=aws-node -o name | while read -r p; do
        echo -n "$p  "
        kubectl -n kube-system get "$p" -o jsonpath='{range .spec.containers[*]}{.name}{" "}{end}'
        echo
     done | tee outputs/11_aws_node_containers.txt
  ```

  {% include step_image.html %}

- {% include step_label.html %} (Sanity) Valida CRD `policyendpoints` (si NetworkPolicy quedó habilitado con VPC CNI).

  ```bash
  kubectl get crd policyendpoints.networking.k8s.aws 2>&1 | tee outputs/11_policyendpoints_crd.txt || true
  ```
  {% include step_image.html %}

#### Tarea 11.2 — Aplicar NetworkPolicy y re-probar IMDS

- {% include step_label.html %} Aplica una NetworkPolicy que permita egress “a todo” **excepto** IMDS (169.254.169.254/32).

  ```bash
  cat > k8s/netpol/01_block_imds.yaml <<'EOF'
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: egress-allow-all-except-imds
    namespace: hardening-insecure
  spec:
    podSelector: {}
    policyTypes: ["Egress"]
    egress:
    - to:
      - ipBlock:
          cidr: 0.0.0.0/0
          except:
          - 169.254.169.254/32
  EOF
  ```
  ```bash
  kubectl apply -f k8s/netpol/01_block_imds.yaml | tee outputs/11_apply_netpol_block_imds.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_INSECURE" get netpol | tee outputs/11_get_netpol.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Reinicia el Deployment `imds-test` y revisa logs (espera bloqueos/timeouts).

  ```bash
  kubectl -n "$NS_INSECURE" delete pod imds-test --ignore-not-found \
    | tee outputs/11_delete_imds_test_pod.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_INSECURE" create deployment imds-test \
    --image=curlimages/curl:8.10.1 \
    -- sh -lc '
      echo "Starting... waiting 15s for NP attach"; sleep 15;
      while true; do
        echo "== $(date) ==";
        echo "-- IMDSv2 token (should TIMEOUT if blocked) --";
        curl -sS -m 2 -i -X PUT "http://169.254.169.254/latest/api/token" \
          -H "X-aws-ec2-metadata-token-ttl-seconds: 60" || true;
        echo;
        sleep 5;
      done
    ' | tee outputs/11_create_imds_test_deploy.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_INSECURE" rollout status deploy/imds-test \
    | tee outputs/11_rollout_imds_after_netpol.txt
  ```
  ```bash
  kubectl -n "$NS_INSECURE" logs deploy/imds-test --tail=80 \
    | tee outputs/11_imds_after_netpol.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura eventos del namespace por si necesitas diagnosticar.

  ```bash
  kubectl -n "$NS_INSECURE" get events --sort-by=.lastTimestamp | tail -n 25 \
    | tee outputs/11_events_after_netpol.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Finalmente elimina el Deploy.
  ```bash
  kubectl -n "$NS_INSECURE" delete deploy imds-test --ignore-not-found | tee outputs/11_delete_imds_test_deploy.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[10] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 12. Secrets: crear y observar el “anti‑mito” base64

#### Tarea 12.1

- {% include step_label.html %} Crea un Secret en el namespace seguro (idempotente).

  ```bash
  kubectl -n "$NS_SECURE" create secret generic app-secret --from-literal=API_KEY="abc-123-SECRET" --dry-run=client -o yaml | kubectl apply -f - | tee outputs/12_apply_secret.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa el YAML (verás base64 en `.data`) y guarda evidencia.

  ```bash
  kubectl -n "$NS_SECURE" get secret app-secret -o yaml | sed -n '1,160p' | tee outputs/12_secret_yaml.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_SECURE" describe secret app-secret | tee outputs/12_secret_describe.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (CKAD) Extrae el valor base64 (codificación, no cifrado) y guarda evidencia.

  ```bash
  kubectl -n "$NS_SECURE" get secret app-secret -o jsonpath='{.data.API_KEY}{""}' | tee outputs/12_secret_base64_value.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[11] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 13. Consumir Secret como volumen read-only en un Pod restricted

#### Tarea 13.1

- {% include step_label.html %} Crea un Pod restricted que monte el Secret como archivos (read-only).

  ```bash
  cat > k8s/secure/04_pod_secret_reader.yaml <<'EOF'
  apiVersion: v1
  kind: Pod
  metadata:
    name: secret-reader
    namespace: hardening-secure
  spec:
    automountServiceAccountToken: false
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      seccompProfile:
        type: RuntimeDefault
    containers:
    - name: reader
      image: busybox:1.36
      command: ["sh","-c","echo '==SECRET=='; cat /secrets/API_KEY; echo; sleep 3600"]
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
      - name: secrets
        mountPath: /secrets
        readOnly: true
      - name: tmp
        mountPath: /tmp
    volumes:
    - name: secrets
      secret:
        secretName: app-secret
    - name: tmp
      emptyDir: {}
  EOF
  ```
  ```bash
  kubectl apply -f k8s/secure/04_pod_secret_reader.yaml | tee outputs/13_apply_secret_reader.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica estado y logs (debe imprimir el secreto).

  ```bash
  kubectl -n "$NS_SECURE" get pod secret-reader -o wide | tee outputs/13_secret_reader_pod.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_SECURE" logs pod/secret-reader | tee outputs/13_secret_reader_logs.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[12] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 14. Validar postura de cifrado en EKS

#### Tarea 14.1

- {% include step_label.html %} Valida versión del clúster y `encryptionConfig` (si existe) y guarda evidencia.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.version" --output text | tee outputs/14_eks_version.txt
  ```
  {% include step_image.html %}
  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.encryptionConfig" --output json | tee outputs/14_eks_encryption_config.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Checklist final rápido (evidencia de estado antes de limpiar).

  ```bash
  kubectl get ns | tee outputs/14_ns_before_cleanup.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 25 | tee outputs/14_events_tail_final.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[13] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 15. Limpieza del laboratorio

Limpiarás recursos (namespaces y, si aplica, clúster).

#### Tarea 15.1

- {% include step_label.html %} Limpia recursos del laboratorio (namespaces).

  ```bash
  kubectl delete ns "$NS_INSECURE" "$NS_SECURE" | tee outputs/14_delete_namespaces.txt
  ```

- {% include step_label.html %} Verifica que ya no existan.

  ```bash
  kubectl get ns | egrep "hardening-(insecure|secure)" || echo "OK: namespaces eliminados" | tee outputs/14_verify_ns_deleted.txt
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
{% capture r1 %}{{ results[14] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}