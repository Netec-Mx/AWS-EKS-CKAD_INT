---
layout: lab
title: "Práctica 19: Monitoreo de consumo energético en EKS con Kepler"
permalink: /lab19/lab19/
images_base: /labs/lab19/img
duration: "75 minutos"
objective:
  - >
    Implementar Kepler en un clúster Amazon EKS para exponer métricas de consumo energético/potencia a nivel nodo/pod/contenedor
    en formato Prometheus, integrarlo con un stack OSS (Prometheus + Grafana), y validar el comportamiento generando carga
    controlada sobre un workload para observar cambios en métricas con verificación y troubleshooting.
prerequisites:
  - "Cuenta AWS con permisos para crear/administrar EKS (EKS, EC2, IAM, VPC, EBS) o un clúster EKS ya disponible para reutilizar."
  - "Windows + Visual Studio Code + Terminal GitBash"
  - "Herramientas: AWS CLI v2, eksctl, kubectl, helm, curl (jq opcional)"
  - "Permisos Kubernetes: crear Namespaces, CRDs (por Helm charts), DaemonSets, Deployments, Services, ServiceMonitors (si usas Prometheus Operator)."
  - "Acceso a Internet para descargar charts e imágenes."
  - "(Recomendado) Managed Node Group Linux (Kepler corre como DaemonSet en nodos Linux)."
introduction:
  - >
    Kepler (Kubernetes-based Efficient Power Level Exporter) expone métricas `kepler_*` para estimar/medir potencia y energía consumida por workloads. En cloud (EKS/EC2) puede no existir acceso a sensores físicos (p.ej., Intel RAPL), por lo que Kepler puede operar en modo estimado/modelado; aun así, es excelente para comparar **tendencias relativas** entre nodos/namespaces/pods y apoyar prácticas de sustentabilidad y optimización.
slug: lab19
lab_number: 19
final_result: >
  Al finalizar tendrás un clúster EKS (creado o reutilizado) con Kepler desplegado como DaemonSet y un stack OSS kube-prometheus-stack: Prometheus Operator + Prometheus + Grafana) consumiendo métricas `kepler_*`. Habrás validado el scraping, ejecutado consultas PromQL útiles y demostrado cambios relativos de potencia/energía al generar carga controlada
  sobre un workload; todo respaldado con evidencia (kubectl/describe/logs/port-forward/curl) y troubleshooting.
notes:
  - "CKAD: práctica muy relevante por el enfoque de verificación/troubleshooting (contexto correcto, port-forward, curl a /metrics, inspección de DaemonSet/Pods, eventos, logs, y consultas rápidas a Prometheus)."
  - "Sustentabilidad: en cloud interpreta métricas como **tendencias relativas** (comparativas), no como medición eléctrica exacta."
  - "Seguridad: Kepler suele requerir `privileged` + `hostPID`. Úsalo en namespaces dedicados y con control de acceso."
  - "Costo: Kepler/Prometheus/Grafana son OSS, pero consumen CPU/RAM/Storage. En EKS pagas control plane + nodos EC2; si Grafana/Prometheus usa PVC, pagas EBS."
references:
  - text: "Kepler (Sustainable Computing) — sitio oficial"
    url: https://sustainable-computing.io/
  - text: "Kepler Helm Chart (repo oficial) — instalación con Helm"
    url: https://sustainable-computing-io.github.io/kepler-helm-chart/
  - text: "Kepler — Métricas (visión general y ejemplos)"
    url: https://sustainable-computing.io/kepler/design/metrics/
  - text: "kube-prometheus-stack (prometheus-community) — chart oficial"
    url: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
  - text: "Prometheus Operator — Custom Resource Definitions (ServiceMonitor/PodMonitor)"
    url: https://github.com/prometheus-operator/prometheus-operator
  - text: "Kepler Operator — Dashboard JSON (para importar en Grafana)"
    url: https://raw.githubusercontent.com/sustainable-computing-io/kepler-operator/v1alpha1/hack/dashboard/assets/kepler/dashboard.json
prev: /lab18/lab18/
next: /lab20/lab20/
---

---

### Costo (resumen)

- **EKS** cobra por clúster por hora (control plane) y nodos (EC2/EBS).
- **Prometheus/Grafana/Kepler** son OSS, pero consumen CPU/RAM/Storage (PVC → EBS).
- Esta práctica NO requiere LoadBalancers externos; usaremos `port-forward`.

> **IMPORTANTE:** crear un clúster EKS genera costo si no lo eliminas.
{: .lab-note .important .compact}

### Convenciones del laboratorio

- **Todo** se ejecuta en **GitBash** dentro de **VS Code**.
- Guarda evidencia en `outputs/` para demostrar resultados.
- Si ya tienes clúster, **omite** Tarea 2.

> **NOTA (CKAD):** disciplina de troubleshooting:
> `get` → `describe` → `events` → `logs` → `exec`/`port-forward`/`curl`.
{: .lab-note .info .compact}

---

### Tarea 1. Preparación del workspace y baseline

Crearás la carpeta del lab, estructura estándar, variables reutilizables y evidencias de herramientas e identidad AWS.

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
  mkdir -p lab19 && cd lab19
  mkdir -p 00-prereqs 01-eks 02-monitoring 03-kepler 04-workload 05-queries outputs logs
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que la estructura quedó creada (evidencia).

  ```bash
  find . -maxdepth 2 -type d | sort | tee outputs/00_dirs.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables base (ajusta región y clúster).

  ```bash
  export AWS_REGION="us-west-2"
  export CLUSTER_NAME="eks-kepler-lab"
  export LAB_ID="$(date +%Y%m%d%H%M%S)"

  export K8S_VERSION="1.33"
  export NODE_TYPE="t3.medium"
  export NODES="2"

  # Namespaces del lab
  export NS_MONITORING="monitoring"
  export NS_KEPLER="kepler"
  export NS_DEMO="energy-demo"
  ```

- {% include step_label.html %} Captura identidad AWS y guarda variables para reuso (reproducible).

  ```bash
  export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  export CALLER_ARN="$(aws sts get-caller-identity --query Arn --output text)"

  cat > outputs/vars.env <<EOF
  AWS_REGION=$AWS_REGION
  CLUSTER_NAME=$CLUSTER_NAME
  LAB_ID=$LAB_ID
  K8S_VERSION=$K8S_VERSION
  NODE_TYPE=$NODE_TYPE
  NODES=$NODES
  NS_MONITORING=$NS_MONITORING
  NS_KEPLER=$NS_KEPLER
  NS_DEMO=$NS_DEMO
  AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID
  CALLER_ARN=$CALLER_ARN
  EOF
  ```
  ```bash
  aws sts get-caller-identity --output json | tee outputs/01_sts_identity.json
  ```
  {% include step_image.html %}
  ```bash
  cat outputs/vars.env | tee outputs/01_vars_echo.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica herramientas.

  ```bash
  aws --version | tee outputs/01_aws_version.txt
  kubectl version --client=true | tee outputs/01_kubectl_version.txt
  eksctl version | tee outputs/01_eksctl_version.txt
  helm version | tee outputs/01_helm_version.txt
  curl --version | head -n 2 | tee outputs/01_curl_version.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear un clúster EKS para el laboratorio (opcional)

Si ya tienes un clúster funcional y `kubectl` apunta al contexto correcto, **puedes omitir** la creación.

> **IMPORTANTE:** elimina recursos al final si este clúster es solo para el lab.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} Crea el clúster con Managed Node Group Linux (2 nodos).

  > **IMPORTANTE:** El cluster tardara aproximadamente **15 minutos** en crearse. Espera el proceso
  {: .lab-note .important .compact}

  ```bash
  export NODEGROUP_NAME="mng-kepler"

  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$K8S_VERSION" \
    --managed \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-type "$NODE_TYPE" \
    --nodes "$NODES" --nodes-min 2 --nodes-max 3 \
    | tee outputs/02_eksctl_create_cluster.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Configura kubeconfig y valida contexto.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION" | tee outputs/02_update_kubeconfig.txt
  ```
  ```bash
  kubectl config current-context | tee outputs/02_kube_context.txt
  ```
  ```bash
  kubectl get nodes -o wide | tee outputs/02_nodes_wide.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Baseline de eventos recientes.

  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 30 | tee outputs/02_events_tail_baseline.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica permisos RBAC para crear namespaces.

  ```bash
  kubectl auth can-i create namespace | tee outputs/02_can_i_create_ns.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Desplegar stack OSS de monitoreo (kube-prometheus-stack) y validar CRDs

Instalarás **kube-prometheus-stack** para obtener Prometheus Operator + Prometheus + Grafana.
Abriremos selectores para descubrir `ServiceMonitors` sin depender de labels del release.

#### Tarea 3.1 — Namespace + values del stack

- {% include step_label.html %} Crea el namespace `monitoring` (idempotente) y captura evidencia.

  ```bash
  kubectl create namespace "$NS_MONITORING" --dry-run=client -o yaml | kubectl apply -f -
  ```
  ```bash
  kubectl get ns "$NS_MONITORING" -o wide | tee outputs/03_ns_monitoring.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el archivo `02-monitoring/values-kps.yaml`.

  > **NOTA:** Muchas instalaciones filtran ServiceMonitors por labels del release; aquí abrimos el selector para que Prometheus “vea” el ServiceMonitor de Kepler sin ajustes extra.
  {: .lab-note .info .compact}

  ```bash
  cat > 02-monitoring/values-kps.yaml <<'EOF'
  prometheus:
    prometheusSpec:
      # Descubrir ServiceMonitors/PodMonitors sin exigir labels del release del chart
      serviceMonitorSelectorNilUsesHelmValues: false
      podMonitorSelectorNilUsesHelmValues: false

  grafana:
    adminPassword: "admin"   # SOLO lab
  EOF
  ```

#### Tarea 3.2 — Instalar kube-prometheus-stack con Helm

- {% include step_label.html %} Agrega repo, actualiza e instala (idempotente).

  ```bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts | tee outputs/03_helm_repo_add.txt
  helm repo update | tee outputs/03_helm_repo_update.txt
  ```
  {% include step_image.html %}
  ```bash
  helm upgrade --install kps prometheus-community/kube-prometheus-stack \
    -n "$NS_MONITORING" --create-namespace \
    -f 02-monitoring/values-kps.yaml \
    | tee outputs/03_helm_install_kps.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera a que levante el stack (captura evidencia de pods Ready).

  ```bash
  kubectl -n "$NS_MONITORING" get pods -o wide | tee outputs/03_monitoring_pods_initial.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_MONITORING" wait --for=condition=Ready pod --all --timeout=600s \
    | tee outputs/03_wait_monitoring_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa el inventario de workloads y servicios del stack de monitoreo.

  ```bash
  kubectl -n "$NS_MONITORING" get deploy,sts,svc | tee outputs/03_monitoring_inventory.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que existan los CRDs de ServiceMonitor/PodMonitor (Prometheus Operator).

  ```bash
  kubectl get crd | egrep "servicemonitors.monitoring.coreos.com|podmonitors.monitoring.coreos.com" \
    | tee outputs/03_monitoring_crds.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Instalar Kepler como DaemonSet y habilitar scraping por Prometheus (ServiceMonitor)

Desplegarás Kepler en nodos Linux y habilitarás `ServiceMonitor` para que Prometheus lo scrappee.

> **Seguridad:** Kepler suele requerir `privileged` + `hostPID`. Mantén el despliegue en un namespace dedicado y úsalo como herramienta de lab/observabilidad.
{: .lab-note .warning .compact}

#### Tarea 4.1 — Namespace + values de Kepler

- {% include step_label.html %} Crea el namespace `kepler` (idempotente) y captura evidencia.

  ```bash
  kubectl create namespace "$NS_KEPLER" --dry-run=client -o yaml | kubectl apply -f -
  ```
  ```bash
  kubectl get ns "$NS_KEPLER" -o wide | tee outputs/04_ns_kepler.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el archivo `03-kepler/values-kepler.yaml`.

  ```bash
  cat > 03-kepler/values-kepler.yaml <<'EOF'
  daemonset:
    hostPID: true
    securityContext:
      privileged: true

  serviceMonitor:
    enabled: true
    interval: 30s
    scrapeTimeout: 10s

  tolerations:
    - operator: Exists

  nodeSelector:
    kubernetes.io/os: linux
  EOF
  ```

#### Tarea 4.2 — Instalar Kepler (Helm chart oficial)

- {% include step_label.html %} Agrega repo oficial e instala (idempotente).

  ```bash
  helm repo add kepler https://sustainable-computing-io.github.io/kepler-helm-chart | tee outputs/04_helm_repo_add_kepler.txt
  helm repo update | tee outputs/04_helm_repo_update_kepler.txt
  ```
  {% include step_image.html %}
  ```bash
  helm upgrade --install kepler kepler/kepler \
    -n "$NS_KEPLER" --create-namespace \
    -f 03-kepler/values-kepler.yaml \
    | tee outputs/04_helm_install_kepler.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica DaemonSet y Service de Kepler en el namespace.

  ```bash
  kubectl -n "$NS_KEPLER" get ds,svc -o wide | tee outputs/04_kepler_ds_svc.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa el estado de los pods de Kepler y en qué nodos están corriendo.

  ```bash
  kubectl -n "$NS_KEPLER" get pods -o wide | tee outputs/04_kepler_pods.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera a que los pods de Kepler estén listos antes de continuar.

  ```bash
  kubectl -n "$NS_KEPLER" wait --for=condition=Ready pod -l app.kubernetes.io/name=kepler --timeout=600s \
    | tee outputs/04_wait_kepler_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma el estado de disponibilidad del DaemonSet de Kepler.

  ```bash
  kubectl -n "$NS_KEPLER" describe daemonset kepler | egrep "Desired Number|Current Number|Ready" \
    | tee outputs/04_kepler_ds_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura las últimas líneas de logs de Kepler para validar operación y detectar errores.

  ```bash
  kubectl -n "$NS_KEPLER" logs -l app.kubernetes.io/name=kepler --tail=20 \
    | tee outputs/04_kepler_logs_tail.txt
  ```

- {% include step_label.html %} Verifica si existe un ServiceMonitor de Kepler para scraping en Prometheus.

  ```bash
  kubectl get servicemonitor -A | grep -i kepler | tee outputs/04_servicemonitor_kepler.txt || true
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Validar métricas de Kepler localmente (/metrics) con port-forward

Primero confirmas que **Kepler expone métricas** en su endpoint. Si aquí falla, todavía NO tiene caso ver Prometheus.

> **NOTA (CKAD):** antes de “culpar” a Prometheus, valida `/metrics` directo con `port-forward` + `curl`.
{: .lab-note .info .compact}

#### Tarea 5.1 — Port-forward + curl

- {% include step_label.html %} En tu **terminal principal**, inicia port-forward al Service de Kepler (deja esta terminal abierta).

  ```bash
  kubectl -n "$NS_KEPLER" port-forward svc/kepler 9102:9102 | tee outputs/05_pf_kepler_9102.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} En una **segunda terminal**, valida métricas y guarda evidencia.

  ```bash
  cd lab19
  curl -s http://localhost:9102/metrics | head -n 25 | tee outputs/05_kepler_metrics_head.txt
  ```
  {% include step_image.html %}
  ```bash
  curl -s http://localhost:9102/metrics | grep -E "^kepler_" | head -n 20 | tee outputs/05_kepler_metrics_any.txt
  ```
  {% include step_image.html %}
  ```bash
  curl -s http://localhost:9102/metrics | grep -E "kepler_.*(watts|joules)" | head -n 40 | tee outputs/05_kepler_metrics_watts_joules.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda un nombre de métrica “real” para usar en consultas posteriores (robusto ante variaciones de nombres).

  > **NOTA:** El endpoint devuelve métricas `kepler_*` y `KEPLER_SAMPLE_METRIC` tiene un nombre (no vacío).
  {: .lab-note .info .compact}

  ```bash
  export KEPLER_SAMPLE_METRIC="$(curl -s http://localhost:9102/metrics | awk '/^kepler_/{print $1; exit}')"
  echo "KEPLER_SAMPLE_METRIC=$KEPLER_SAMPLE_METRIC" | tee outputs/05_kepler_sample_metric.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Verificar scraping en Prometheus (targets + API rápida)

Aquí confirmas que Prometheus **ingiere series** de Kepler.

#### Tarea 6.1 — Port-forward a Prometheus + consulta API

- {% include step_label.html %} Continua la ejecución en la **segunda terminal** e identifica el Service de Prometheus del stack y guárdalo en una variable.

  ```bash
  source outputs/vars.env
  kubectl -n "$NS_MONITORING" get svc \
    -o custom-columns=NAME:.metadata.name,PORTS:.spec.ports[*].port --no-headers \
    | awk '$2 ~ /9090/ {print $1}' \
    | tee outputs/06_prom_svc_9090_candidates.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define `PROM_SVC` (ajusta si tu salida es distinta) e inicia port-forward (deja la terminal abierta).

  ```bash
  export PROM_SVC="kps-kube-prometheus-stack-prometheus"
  ```
  ```bash
  kubectl -n "$NS_MONITORING" port-forward svc/"$PROM_SVC" 9090:9090 \
  | tee outputs/06_prom_port_forward.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre una **tercera terminal** para consultar la API de Prometheus.

  ```bash
  cd lab19
  source outputs/vars.env
  curl -sG --data-urlencode "query=up" http://localhost:9090/api/v1/query | head -n 20 | tee outputs/06_prom_up.json
  ```
  {% include step_image.html %}
  ```bash
  export KEPLER_SAMPLE_METRIC="$(curl -s http://localhost:9102/metrics | awk '/^kepler_/{print $1; exit}')"
  curl -sG --data-urlencode "query=$KEPLER_SAMPLE_METRIC" http://localhost:9090/api/v1/query | head -n 20 | tee outputs/06_prom_kepler_sample_metric.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Verifica targets de Kepler (útil cuando hay problemas de ServiceMonitor).

  ```bash
  curl -s http://localhost:9090/api/v1/targets \
    | jq -r '
      .data.activeTargets[]
      | select((.labels.job // "" | test("kepler"; "i")) or (.scrapePool // "" | test("kepler"; "i")))
      | {job:(.labels.job // .scrapePool), instance:.labels.instance, health:.health, scrapeUrl:.scrapeUrl, lastScrape:.lastScrape, lastError:.lastError}
    ' \
    | tee outputs/06_prom_targets_kepler_min.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que la consulta a Prometheus haya respondido con status: success.

  > **NOTA:** Prometheus responde `success` y regresa series para al menos una métrica `kepler_*`.
  {: .lab-note .info .compact}

  ```bash
  grep -o '"status":"[^"]*"' outputs/06_prom_kepler_sample_metric.json | head -n 1 | tee outputs/06_prom_status_check.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Generar carga controlada y observar cambios relativos (PromQL / API)

Desplegarás un workload CPU-bound y observarás cambios relativos en métricas (no “watts exactos” en cloud).

#### Tarea 7.1 — Workload de carga (cpu-burn)

- {% include step_label.html %} Continua en la **tercera terminal**  y crea el namespace de demo (idempotente).

  ```bash
  kubectl create namespace "$NS_DEMO" --dry-run=client -o yaml | kubectl apply -f -
  kubectl get ns "$NS_DEMO" -o wide | tee outputs/07_ns_demo.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea y aplica el workload (Deployment).

  ```bash
  cat > 04-workload/10_cpu_burn.yaml <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: cpu-burn
    namespace: energy-demo
    labels:
      app: cpu-burn
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: cpu-burn
    template:
      metadata:
        labels:
          app: cpu-burn
      spec:
        containers:
        - name: burner
          image: busybox:1.36
          command: ["sh","-c"]
          args:
            - >
              echo "Burning CPU..." ;
              while true; do
                dd if=/dev/zero of=/dev/null bs=1M count=1024 2>/dev/null;
              done
          resources:
            requests:
              cpu: "250m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "128Mi"
  EOF
  ```

  ```bash
  kubectl apply -f 04-workload/10_cpu_burn.yaml | tee outputs/07_apply_cpu_burn.txt
  ```
  ```bash
  kubectl -n "$NS_DEMO" rollout status deploy/cpu-burn | tee outputs/07_rollout_cpu_burn.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS_DEMO" get pods -o wide | tee outputs/07_demo_pods.txt
  ```
  {% include step_image.html %}

#### Tarea 7.2 — PromQL recomendado + prueba rápida por API (top pods)

- {% include step_label.html %} Crea guía PromQL del lab.

  ```bash
  cat > 05-queries/promql.md <<'EOF'
  # PromQL — Kepler (lab19)

  > Kepler exporta energía acumulada en Joules (`*_joules_total`).
  > Para obtener "Watts" (potencia) usa `rate( ...[1m])` => Joules/segundo = Watts.

  ## 1) Baseline (¿hay series?)
  # ¿Kepler está publicando datos por nodo?
  count(kepler_node_platform_joules_total)

  # ¿Kepler está publicando datos por contenedor?
  count(kepler_container_joules_total)

  # Ver algunas series (útil para validar labels)
  kepler_container_joules_total
  kepler_node_platform_joules_total


  ## 2) Potencia por nodo (Watts aprox)
  # Potencia total plataforma por nodo
  rate(kepler_node_platform_joules_total[1m])

  # Potencia por dominio (útil para comparar)
  rate(kepler_node_package_joules_total[1m])
  rate(kepler_node_core_joules_total[1m])
  rate(kepler_node_dram_joules_total[1m])
  rate(kepler_node_uncore_joules_total[1m])

  # Total cluster (suma de nodos)
  sum(rate(kepler_node_platform_joules_total[1m]))


  ## 3) Potencia por pod (Watts) usando métricas de contenedor
  # Potencia total agregada por pod (todas las contribuciones: package/core/dram/other/uncore)
  sum by (container_namespace, pod_name) (
    rate(kepler_container_joules_total[1m])
  )

  # Variante "CPU package" por pod (si quieres aproximar CPU/PKG)
  sum by (container_namespace, pod_name) (
    rate(kepler_container_package_joules_total[1m])
  )

  # Variante DRAM por pod
  sum by (container_namespace, pod_name) (
    rate(kepler_container_dram_joules_total[1m])
  )


  ## 4) Top pods por consumo (Watts)
  # Top 10 pods por potencia total
  topk(10,
    sum by (container_namespace, pod_name) (
      rate(kepler_container_joules_total[1m])
    )
  )

  # Top 10 pods por "CPU package"
  topk(10,
    sum by (container_namespace, pod_name) (
      rate(kepler_container_package_joules_total[1m])
    )
  )


  ## 5) Energía consumida (Joules) en una ventana
  # Energía por pod en los últimos 5 minutos
  sum by (container_namespace, pod_name) (
    increase(kepler_container_joules_total[5m])
  )

  # Top 10 por energía en 5 minutos
  topk(10,
    sum by (container_namespace, pod_name) (
      increase(kepler_container_joules_total[5m])
    )
  )
  EOF
  ```

- {% include step_label.html %} **OPCIONAL** Puedes probar cada una de las consultas directamene en Prometheus abre una pestaña en el navegador **`http://localhost:9090`** y prueba las consultas

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. (Opcional) Grafana: port-forward + importar dashboard Kepler

Esta tarea es opcional (visual). Si te quedas solo con Prometheus, la práctica ya está completa a nivel técnico.

#### Tarea 8.1 — Port-forward a Grafana

- {% include step_label.html %} Continua en la **tercera terminal** e identifica el Service de Grafana y haz port-forward.

  ```bash
  kubectl -n "$NS_MONITORING" get svc | grep -i grafana | tee outputs/08_grafana_svc_list.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define `GRAFANA_SVC` (ajusta si difiere) y ejecuta port-forward (deja terminal abierta).

  ```bash
  export GRAFANA_SVC="${GRAFANA_SVC:-kps-grafana}"
  kubectl -n "$NS_MONITORING" port-forward svc/"$GRAFANA_SVC" 3000:80
  ```
  {% include step_image.html %}

#### Tarea 8.2 — Importar dashboard Kepler

- {% include step_label.html %} En una **cuarta terminal** ejecuta el siguiente comando para crear el dashboard de grafana

  {% raw %}
  ```bash
  cd lab19
  source outputs/vars.env  
  cat > 05-queries/kepler-dashboard-lab19.json <<'EOF'
  {
    "__requires": [
      { "type": "grafana", "id": "grafana", "name": "Grafana", "version": "9.0.0" },
      { "type": "datasource", "id": "prometheus", "name": "Prometheus", "version": "1.0.0" },
      { "type": "panel", "id": "timeseries", "name": "Time series", "version": "" },
      { "type": "panel", "id": "table", "name": "Table", "version": "" }
    ],
    "annotations": {
      "list": [
        {
          "builtIn": 1,
          "datasource": "-- Grafana --",
          "enable": true,
          "hide": true,
          "iconColor": "rgba(0, 211, 255, 1)",
          "name": "Annotations & Alerts",
          "type": "dashboard"
        }
      ]
    },
    "editable": true,
    "graphTooltip": 0,
    "id": null,
    "panels": [
      {
        "type": "timeseries",
        "title": "Potencia del clúster (W) — Plataforma",
        "datasource": "Prometheus",
        "targets": [
          {
            "refId": "A",
            "expr": "sum(rate(kepler_node_platform_joules_total[$__rate_interval]))",
            "legendFormat": "cluster",
            "range": true,
            "instant": false
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 }
      },
      {
        "type": "timeseries",
        "title": "Potencia por nodo (W) — Plataforma",
        "datasource": "Prometheus",
        "targets": [
          {
            "refId": "A",
            "expr": "rate(kepler_node_platform_joules_total[$__rate_interval])",
            "legendFormat": "{{instance}}",
            "range": true,
            "instant": false
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 }
      },
      {
        "type": "table",
        "title": "Top Pods por potencia (W) — Total (instantáneo)",
        "datasource": "Prometheus",
        "targets": [
          {
            "refId": "A",
            "format": "table",
            "expr": "topk(10, sum by (container_namespace, pod_name) (rate(kepler_container_joules_total[$__rate_interval])))",
            "instant": true,
            "range": false
          }
        ],
        "options": {
          "showHeader": true,
          "sortBy": [
            { "displayName": "Value", "desc": true }
          ]
        },
        "gridPos": { "h": 10, "w": 12, "x": 0, "y": 8 }
      },
      {
        "type": "table",
        "title": "Top Pods por energía (J) — últimos 5m (instantáneo)",
        "datasource": "Prometheus",
        "targets": [
          {
            "refId": "A",
            "format": "table",
            "expr": "topk(10, sum by (container_namespace, pod_name) (increase(kepler_container_joules_total[5m])))",
            "instant": true,
            "range": false
          }
        ],
        "options": {
          "showHeader": true,
          "sortBy": [
            { "displayName": "Value", "desc": true }
          ]
        },
        "gridPos": { "h": 10, "w": 12, "x": 12, "y": 8 }
      }
    ],
    "refresh": "10s",
    "schemaVersion": 39,
    "tags": [ "kepler", "lab19", "joules-to-watts" ],
    "templating": { "list": [] },
    "time": { "from": "now-15m", "to": "now" },
    "timezone": "browser",
    "title": "Kepler • Lab19 — Energía y Potencia (J → W)",
    "uid": "kepler-lab19",
    "version": 2
  }
  EOF
  ```
  {% endraw %}

- {% include step_label.html %} Abre la URL en el navegador Chrome/Edge y coloca las credenciales **`admin/admin`**, si te pide reconfirmar contrasela vuelve a colocar **`admin`**

  ```bash
  http://localhost:3000
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora en la interfaz de Grafana e importa el dashboard de Kepler en Grafana: **Dashboards → Import**

  {% include step_image.html %}

- {% include step_label.html %} Ahora carga el archivo que se encuentra en el directorio de la practica **05-queries/kepler-dashboard-lab19.json**

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Import**

- {% include step_label.html %} Finalmente observaras las metricas en el dashboard de Grafana.

  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 9. Troubleshooting 

Checklist final tipo CKAD

#### Tarea 9.1 — Checklist CKAD

- {% include step_label.html %} Continua en la **cuarta terminal** y revisa inventario y eventos. Si no hay errores solo ejecuta los comandos.

  ```bash
  kubectl -n "$NS_KEPLER" get ds,po,svc -o wide | tee outputs/09_kepler_inventory.txt
  ```
  ```bash
  kubectl -n "$NS_MONITORING" get deploy,sts,svc -o wide | tee outputs/09_monitoring_inventory.txt
  ```
  ```bash
  kubectl -n "$NS_DEMO" get deploy,po -o wide | tee outputs/09_demo_inventory.txt
  ```
  ```bash
  kubectl -n "$NS_KEPLER" get events --sort-by=.lastTimestamp | tail -n 30 | tee outputs/09_kepler_events_tail.txt
  ```
  ```bash
  kubectl -n "$NS_MONITORING" get events --sort-by=.lastTimestamp | tail -n 30 | tee outputs/09_monitoring_events_tail.txt
  ```

- {% include step_label.html %} Logs útiles (si algo salió mal).

  ```bash
  kubectl -n "$NS_KEPLER" logs -l app.kubernetes.io/name=kepler --tail=120 | tee outputs/09_kepler_logs_tail.txt || true
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 10. Limpieza 

Eliminación de namespaces para no dejar cargas residuales. (Opcional: borrar clúster si lo creaste solo para el lab).

#### Tarea 10.1

- {% include step_label.html %} Elimina namespaces del lab (ahorro de costo).

  ```bash
  kubectl delete ns "$NS_DEMO" "$NS_KEPLER" "$NS_MONITORING" --ignore-not-found | tee outputs/09_delete_namespaces.txt
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

- {% include step_label.html %} Corta los procesos de las otras 3 terminales con **`CTRL + C`** y cierralas todas antes de avanzar al siguiente laboratorio.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[9] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}