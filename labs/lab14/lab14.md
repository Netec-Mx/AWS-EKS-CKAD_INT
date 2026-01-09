---
layout: lab
title: "Práctica 14: Configuración de despliegues con alta disponibilidad en EKS"
permalink: /lab14/lab14/
images_base: /labs/lab14/img
duration: "60 minutos"
objective:
  - "Diseñar y desplegar en Amazon EKS una aplicación altamente disponible (HA) aplicando patrones Kubernetes: réplicas + RollingUpdate controlado, probes, topology spread constraints (multi‑AZ), pod anti‑affinity (multi‑nodo) y PodDisruptionBudget (PDB) para proteger contra disrupciones voluntarias; finalmente, validar la resiliencia con pruebas de tráfico, un rollout y un drain de nodo con troubleshooting estilo CKAD."
prerequisites:
  - "Amazon EKS accesible con kubectl"
  - "Clúster EKS con nodos en ≥2 AZ (recomendado para demostrar HA real)"
  - "Windows + Visual Studio Code + Terminal GitBash"
  - "Herramientas: AWS CLI v2, kubectl"
  - "(Opcional) eksctl (si vas a crear un clúster para labs)"
  - "(Recomendado) git"
  - "(Opcional) jq (para conteos/salidas limpias)"
  - "Permisos Kubernetes: crear Namespace, Deployment, Service, PodDisruptionBudget, y ejecutar cordon/drain (nodos)"
introduction: 
  "En Kubernetes, “alta disponibilidad” no es solo “tener varias réplicas”: también es evitar que todas queden en el mismo nodo/AZ, asegurar que el Service solo enrute a Pods listos (readiness), controlar el impacto durante actualizaciones (RollingUpdate) y proteger contra mantenimiento planificado (drain/evictions) con PDB. En EKS, AWS recomienda data plane multi‑AZ y usar labels bien conocidos (como topology.kubernetes.io/zone) con topology spread constraints para distribuir Pods en dominios de falla."
slug: lab14
lab_number: 14
final_result: |
  Al finalizar tendrás un despliegue en Amazon EKS con alta disponibilidad operativa: 
  - (1) Service + Deployment con probes que controlan endpoints listos.
  - (2) RollingUpdate con maxUnavailable/maxSurge para mantener disponibilidad durante upgrades. 
  - (3) Distribución por AZ y por nodo mediante topology spread constraints y anti‑affinity.
  - (4) Protección ante disrupciones voluntarias con PDB; todo validado con tráfico real, un rollout y un drain de nodo con evidencia en logs, describe y events (enfoque CKAD).
notes:
  - "CKAD: práctica núcleo (Deployments/Services, probes, strategy/rollouts, scheduling avanzado, PDB, y troubleshooting con kubectl get/describe/events/logs)."
  - "Este laboratorio usa Service ClusterIP + Pods de prueba internos (curl) para evitar costos y ruido de LoadBalancers/Ingress."
  - "Multi‑AZ real: si tus nodos están en una sola AZ, la HA queda limitada (solo multi‑nodo dentro de esa AZ)."
references:
  - text: "AWS EKS Best Practices Guides - Reliability / Multi-AZ considerations"
    url: https://docs.aws.amazon.com/eks/latest/best-practices/reliability.html
  - text: "Kubernetes - Deployments (RollingUpdate, maxUnavailable/maxSurge)"
    url: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
  - text: "Kubernetes - Probes (liveness/readiness/startup)"
    url: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
  - text: "Kubernetes - Topology Spread Constraints"
    url: https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/
  - text: "Kubernetes - Assigning Pods to Nodes (Affinity/Anti-affinity)"
    url: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
  - text: "Kubernetes - Pod Disruption Budgets"
    url: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
  - text: "Kubernetes - Safely Drain a Node (kubectl drain)"
    url: https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/
prev: /lab13/lab13/
next: /lab15/lab15/
---


---

### Tarea 1. Preparar la carpeta de la práctica y el entorno (baseline multi‑AZ)

Crearás la carpeta del laboratorio, abrirás **GitBash** dentro de **VS Code**, definirás variables y validarás que `aws` y `kubectl` apuntan a la cuenta y clúster correctos. Además, confirmarás que los nodos tienen labels de zona/hostname para poder aplicar *topology spread constraints*.

> **NOTA (CKAD):** Aquí practicas el hábito “de examen”: validar contexto, permisos y señales del clúster **antes** de tocar YAML (`kubectl config`, `kubectl auth can-i`, `kubectl get/describe`, `kubectl get events`).
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
  mkdir -p lab14 && cd lab14
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea la estructura estándar de carpetas.

  ```bash
  mkdir -p k8s tests scripts outputs
  ```

- {% include step_label.html %} Confirma la estructura creada.

  ```bash
  find . -maxdepth 2 -type d | sort | tee outputs/00_dirs.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables base (ajusta a tu entorno).

  ```bash
  export AWS_REGION="us-west-2"
  export NS="ha-lab"
  export APP_NAME="web"
  ```

- {% include step_label.html %} Valida identidad de AWS y región (reduce el riesgo de operar en otra cuenta/región).

  ```bash
  aws sts get-caller-identity | tee outputs/01_sts_identity.json
  ```
  {% include step_image.html %}
  ```bash
  aws configure get region || true
  echo "AWS_REGION=$AWS_REGION"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. (Opcional) Crear un clúster EKS multi‑AZ para el laboratorio

**Omite esta tarea** si ya tienes un clúster funcional y multi‑AZ.

> **IMPORTANTE:** Si creas un clúster solo para laboratorio, elimínalo al final.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} Verifica versiones soportadas de EKS en tu región antes de elegir `K8S_VERSION`.

  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```

- {% include step_label.html %} Obtén al menos 2 AZ disponibles en tu región (para multi‑AZ real).

  ```bash
  aws ec2 describe-availability-zones --region "$AWS_REGION" --query "AvailabilityZones[?State=='available'].ZoneName" --output text | tee outputs/15_az_list.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables del clúster (ajusta si tu región tiene AZs con sufijos distintos).

  ```bash
  export CLUSTER_NAME="eks-ha-lab"
  export NODEGROUP_NAME="mng-ha"
  export K8S_VERSION="1.33"
  export ZONES="$(awk '{print $1","$2}' outputs/15_az_list.txt)"
  ```
  ```bash
  echo "ZONES=$ZONES" | tee outputs/16_zones_selected.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el clúster con Managed Node Group (≥3 nodos recomendado para ver spread + PDB con comodidad).

  ```bash
  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$K8S_VERSION" \
    --managed --zones "$ZONES" \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-type t3.medium --nodes 3 --nodes-min 3 --nodes-max 5 | tee outputs/17_eksctl_create_cluster.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Configura `kubeconfig` y valida multi‑AZ (labels de zona visibles).

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}
  ```bash
  kubectl get nodes -L topology.kubernetes.io/zone -o wide | tee outputs/18_nodes_multi_az.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica estado de nodos y captura evidencia (deben estar `Ready`).

  ```bash
  kubectl get nodes -o wide | tee outputs/05_nodes_wide.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma labels de topología necesarias (zona y hostname).

  > **NOTA:** Debes ver una columna `topology.kubernetes.io/zone` poblada (idealmente con ≥2 zonas). Si está vacía, tu clúster no podrá demostrar HA por AZ.
  {: .lab-note .important .compact}

  ```bash
  kubectl get nodes -L topology.kubernetes.io/zone -L kubernetes.io/hostname | tee outputs/06_nodes_topology_labels.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional recomendado) Resume cuántos nodos tienes por AZ (evidencia rápida de multi‑AZ).

  ```bash
  kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.labels.topology\.kubernetes\.io/zone}{"\n"}{end}' | sort | uniq -c | tee outputs/07_nodes_per_zone.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida permisos mínimos necesarios para la práctica (cordon/drain requiere privilegios).

  > **NOTA:** respuestas `yes` en lo esencial. Si `patch node` devuelve `no`, no podrás ejecutar `cordon/drain` (Tarea 6).
  {: .lab-note .important .compact}

  ```bash
  kubectl auth can-i create namespace | tee outputs/08_can_i_ns.txt
  ```
  ```bash
  kubectl auth can-i create poddisruptionbudget -n default | tee outputs/09_can_i_pdb.txt
  ```
  ```bash
  kubectl auth can-i patch node | tee outputs/10_can_i_patch_node.txt
  ```
  ```bash
  kubectl auth can-i create deployment -n default | tee outputs/11_can_i_deploy.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el namespace del laboratorio (idempotente).

  ```bash
  kubectl create ns "$NS" --dry-run=client -o yaml | kubectl apply -f -
  ```
  ```bash
  kubectl get ns "$NS" | tee outputs/12_ns_created.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura baseline de eventos recientes (para comparar si algo se rompe después).

  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 20 | tee outputs/13_events_tail_baseline.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Deploy base: Service + Deployment con probes y RollingUpdate controlado

Desplegarás una app *stateless* con **3 réplicas** y un **Service ClusterIP**. Configurarás **readiness/liveness** para que Kubernetes no envíe tráfico a Pods no listos y ajustarás **RollingUpdate** (`maxUnavailable`, `maxSurge`) para mantener disponibilidad durante actualizaciones.

> **NOTA (CKAD):** Esto es núcleo CKAD: Deployments, Services, labels/selectors, strategy/rollouts, probes y validación con endpoints.
{: .lab-note .info .compact}

#### Tarea 3.1

- {% include step_label.html %} Crea el manifiesto base de la app (Service + Deployment).

  ```bash
  cat > k8s/01-app-base.yaml <<'EOF'
  apiVersion: v1
  kind: Service
  metadata:
    name: web
    namespace: ha-lab
    labels:
      app: web
  spec:
    selector:
      app: web
    ports:
    - name: http
      port: 80
      targetPort: 9898
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: web
    namespace: ha-lab
    labels:
      app: web
  spec:
    replicas: 3
    revisionHistoryLimit: 5
    strategy:
      type: RollingUpdate
      rollingUpdate:
        # Mantén disponibilidad durante el rollout:
        maxUnavailable: 1
        maxSurge: 1
    selector:
      matchLabels:
        app: web
    template:
      metadata:
        labels:
          app: web
          tier: frontend
      spec:
        terminationGracePeriodSeconds: 20
        containers:
        - name: web
          image: ghcr.io/stefanprodan/podinfo:6.7.1
          ports:
          - containerPort: 9898
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 300m
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /readyz
              port: 9898
            initialDelaySeconds: 3
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /healthz
              port: 9898
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
  EOF
  ```

- {% include step_label.html %} Aplica el manifiesto y espera el rollout.

  ```bash
  kubectl apply -f k8s/01-app-base.yaml | tee outputs/20_apply_app_base.txt
  ```
  ```bash
  kubectl -n "$NS" rollout status deploy/"$APP_NAME" | tee outputs/21_rollout_status_base.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica Pods y endpoints listos (readiness controla endpoints).

  > **NOTA:** Debes ver **3 Pods Running/Ready** y endpoints con **3 IPs** (o 3 entradas) listos.
  {: .lab-note .important .compact}

  ```bash
  kubectl -n "$NS" get pods -o wide | tee outputs/22_pods_wide_base.txt
  ```
  ```bash
  kubectl -n "$NS" get endpoints "$APP_NAME" -o wide | tee outputs/23_endpoints_base.txt
  ```
  {% include step_image.html %}

#### Tarea 3.2 (cliente interno para pruebas)

- {% include step_label.html %} Crea un Pod de pruebas (curl) interno (sin LoadBalancer/Ingress).

  ```bash
  kubectl -n "$NS" run curl --image=curlimages/curl:8.10.1 --restart=Never --command -- sleep 3600 | tee outputs/24_create_curl_pod.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" wait --for=condition=Ready pod/curl --timeout=120s | tee outputs/25_wait_curl_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba el Service desde dentro del clúster y guarda evidencia.

  ```bash
  kubectl -n "$NS" exec -it curl -- sh -lc 'curl -sS http://web/healthz && echo && curl -sS http://web/readyz && echo' | tee outputs/26_curl_health_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura evidencia del Deployment (incluye strategy, probes, recursos).

  ```bash
  kubectl -n "$NS" describe deploy "$APP_NAME" | sed -n '1,220p' | tee outputs/27_describe_deploy_base.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. HA de scheduling: topology spread constraints (multi‑AZ + multi‑nodo) y anti‑affinity

Evitarás el anti‑patrón más común: *todas las réplicas en el mismo nodo o AZ*. Aplicarás:
- **TopologySpreadConstraints** por `topology.kubernetes.io/zone` (AZ) y por `kubernetes.io/hostname` (nodo).
- **Pod anti‑affinity** para preferir separación por nodo.

> **NOTA (CKAD):** scheduling avanzado suele fallar con Pods `Pending`. Tu evidencia siempre es `kubectl describe pod` + `events`.
{: .lab-note .info .compact}

#### Tarea 4.1

- {% include step_label.html %} Crea el manifiesto del Deployment con constraints/affinity (solo Deployment; el Service ya existe).

  ```bash
  cat > k8s/02-deploy-ha-scheduling.yaml <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: web
    namespace: ha-lab
    labels:
      app: web
  spec:
    replicas: 3
    revisionHistoryLimit: 5
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 1
        maxSurge: 1
    selector:
      matchLabels:
        app: web
    template:
      metadata:
        labels:
          app: web
          tier: frontend
      spec:
        terminationGracePeriodSeconds: 20
        topologySpreadConstraints:
        # 1) Spread por AZ (si hay ≥2 AZ disponibles)
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: web
        # 2) Spread por nodo (hostname)
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: web
        affinity:
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: web
                topologyKey: kubernetes.io/hostname
        containers:
        - name: web
          image: ghcr.io/stefanprodan/podinfo:6.7.1
          ports:
          - containerPort: 9898
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 300m
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /readyz
              port: 9898
            initialDelaySeconds: 3
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /healthz
              port: 9898
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
  EOF
  ```

- {% include step_label.html %} Aplica el manifiesto (esto dispara un nuevo ReplicaSet) y espera el rollout.

  ```bash
  kubectl apply -f k8s/02-deploy-ha-scheduling.yaml | tee outputs/30_apply_ha_scheduling.txt
  ```
  ```bash
  kubectl -n "$NS" rollout status deploy/"$APP_NAME" | tee outputs/31_rollout_status_ha_scheduling.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica distribución por nodo y AZ (evidencia).

  ```bash
  kubectl -n "$NS" get pods -l app=web -o wide | tee outputs/32_pods_wide_after_scheduling.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Genera una tabla Pod → Node → Zone (evidencia clara de HA).

  > **NOTA:** 3 Pods en **3 nodos distintos**. Si tu clúster es multi‑AZ, deberías ver Pods distribuidos en **≥2 AZ**.
  {: .lab-note .important .compact}

  ```bash
  kubectl -n "$NS" get pod -l app=web -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'     | while read -r pod node; do
        zone="$(kubectl get node "$node" -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}')"
        echo -e "${pod}\t${node}\t${zone}"
      done | column -t | tee outputs/33_pod_node_zone_table.txt
  ```
  {% include step_image.html %}

#### Tarea 4.2 Troubleshooting (si algún Pod queda `Pending`)

- {% include step_label.html %} Identifica Pods `Pending` y captura evidencia con `describe`.

  ```bash
  kubectl -n "$NS" get pods | tee outputs/34_pods_status_if_pending.txt
  ```
  ```bash
  # Reemplaza POD_PENDING por el nombre real si aplica:
  kubectl -n "$NS" describe pod POD_PENDING | sed -n '1,220p' | tee outputs/35_describe_pending_pod.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa eventos recientes del namespace (suele explicar restricciones/capacidad).

  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 40 | tee outputs/36_events_tail_after_scheduling.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. PodDisruptionBudget (PDB) + graceful shutdown

Protegerás disponibilidad durante **disrupciones voluntarias** (ej. `kubectl drain` por mantenimiento). Crearás un **PDB** que permita como máximo **1 pod indisponible** y agregarás un `preStop` para un shutdown más “limpio” (reduce cortes abruptos).

> **NOTA (CKAD):** PDB aplica a *evictions voluntarias*; no evita fallas involuntarias (ej. caída súbita de un nodo).
{: .lab-note .info .compact}

#### Tarea 5.1 (PDB)

- {% include step_label.html %} Crea el manifiesto del PDB.

  ```bash
  cat > k8s/03-pdb-web.yaml <<'EOF'
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata:
    name: web-pdb
    namespace: ha-lab
  spec:
    maxUnavailable: 1
    selector:
      matchLabels:
        app: web
  EOF
  ```

- {% include step_label.html %} Aplica el PDB y revisa su estado (allowed disruptions).

  > **NOTA:** `Allowed disruptions` debe ser **≥ 1** (normalmente 1 con 3 réplicas y maxUnavailable=1).
  {: .lab-note .important .compact}

  ```bash
  kubectl apply -f k8s/03-pdb-web.yaml | tee outputs/40_apply_pdb.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get pdb | tee outputs/41_get_pdb.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" describe pdb web-pdb | sed -n '1,220p' | tee outputs/42_describe_pdb.txt
  ```
  {% include step_image.html %}

#### Tarea 5.2 (preStop)

- {% include step_label.html %} Agrega `preStop` al contenedor (simula shutdown controlado con `sleep 5`).

  ```bash
  kubectl -n "$NS" patch deploy "$APP_NAME" --type='json' -p='[
    {"op":"add","path":"/spec/template/spec/containers/0/lifecycle","value":{
      "preStop":{"exec":{"command":["sh","-c","sleep 5"]}}
    }}
  ]' | tee outputs/43_patch_prestop.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera el rollout disparado por el cambio en el Pod template.

  ```bash
  kubectl -n "$NS" rollout status deploy/"$APP_NAME" | tee outputs/44_rollout_after_prestop.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que `lifecycle.preStop` quedó aplicado (evidencia).

  ```bash
  kubectl -n "$NS" get deploy "$APP_NAME" -o jsonpath='{.spec.template.spec.containers[0].lifecycle}{"\n"}' | tee outputs/45_verify_lifecycle.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Pruebas de resiliencia: tráfico continuo + rollout + drain

Generarás tráfico constante para detectar interrupciones, ejecutarás un rolling update (cambio de imagen) y realizarás un drain de un nodo para verificar que:
1) el Service sigue respondiendo,
2) el rollout respeta `maxUnavailable/maxSurge`, y
3) el PDB evita disrupciones excesivas.

> **NOTA (CKAD):** Si “algo no cuadra”, tu secuencia siempre es: `get` → `describe` → `events` → `logs`.
{: .lab-note .info .compact}

#### Tarea 6.1 (tráfico continuo)

- {% include step_label.html %} Crea un Pod tester que haga requests en loop (monitor casero).

  ```bash
  cat > k8s/04-tester.yaml <<'EOF'
  apiVersion: v1
  kind: Pod
  metadata:
    name: tester
    namespace: ha-lab
    labels:
      app: tester
  spec:
    restartPolicy: Never
    containers:
    - name: tester
      image: curlimages/curl:8.10.1
      command: ["sh","-c"]
      args:
        - |
          while true; do
            code=$(curl -s -o /dev/null -w "%{http_code}" http://web/healthz);
            echo "$(date) http_code=$code";
            sleep 1;
          done
  EOF
  ```

- {% include step_label.html %} Aplica el tester y espera Ready.

  ```bash
  kubectl apply -f k8s/04-tester.yaml | tee outputs/50_apply_tester.txt
  ```
  ```bash
  kubectl -n "$NS" wait --for=condition=Ready pod/tester --timeout=120s | tee outputs/51_wait_tester_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa logs del tester (debe imprimir `http_code=200` continuamente).

  ```bash
  kubectl -n "$NS" logs -f pod/tester
  ```
  {% include step_image.html %}

#### Tarea 6.2 (RollingUpdate controlado)

- {% include step_label.html %} En una **segunda terminal**, actualiza la imagen y valida rollout + history.

  ```bash
  cd lab14
  export AWS_REGION="us-west-2"
  export NS="ha-lab"
  export APP_NAME="web"
  ```
  ```bash
  kubectl -n "$NS" set image deploy/"$APP_NAME" web=ghcr.io/stefanprodan/podinfo:latest | tee outputs/52_set_image_rollout.txt
  ```
  ```bash
  kubectl -n "$NS" rollout status deploy/"$APP_NAME" | tee outputs/53_rollout_status_after_update.txt
  ```
  ```bash
  kubectl -n "$NS" rollout history deploy/"$APP_NAME" | tee outputs/54_rollout_history.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Continua en la **segunda terminal** para capturar el estado de Pods durante/tras el rollout.

  > **NOTA:** El tester sigue mostrando `http_code=200` (o como máximo micro‑blips). El rollout finaliza y endpoints permanecen con Pods listos.
  {: .lab-note .important .compact}
  
  ```bash
  kubectl -n "$NS" get rs -o wide | tee outputs/55_rs_wide_after_rollout.txt
  ```
  ```bash
  kubectl -n "$NS" get pods -o wide | tee outputs/56_pods_wide_after_rollout.txt
  ```
  ```bash
  kubectl -n "$NS" get endpoints "$APP_NAME" -o wide | tee outputs/57_endpoints_after_rollout.txt
  ```
  {% include step_image.html %}

#### Tarea 6.3 (drain de nodo con PDB)

- {% include step_label.html %} Identifica un nodo donde esté corriendo al menos 1 Pod de `web` (sin “inventar” el nombre).

  ```bash
  kubectl -n "$NS" get pods -l app=web -o wide | tee outputs/58_web_pods_before_drain.txt
  ```
  ```bash
  export NODE_NAME="$(kubectl -n "$NS" get pod -l app=web -o jsonpath='{.items[0].spec.nodeName}')"
  echo "NODE_NAME=$NODE_NAME" | tee outputs/59_selected_node.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Cordona el nodo (evita nuevos pods en ese nodo).

  ```bash
  kubectl cordon "$NODE_NAME" | tee outputs/60_cordon_node.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Drena el nodo (evictions voluntarias).

  > **NOTA:** Kubernetes evicta Pods del nodo (respetando el PDB). Si el drain se “pausa” por el PDB, eso significa que tu presupuesto está protegiendo disponibilidad (y debes revisar distribución de Pods).
  {: .lab-note .important .compact}

  ```bash
  kubectl drain "$NODE_NAME" --ignore-daemonsets --delete-emptydir-data | tee outputs/61_drain_node.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica la reubicación de Pods y estado del PDB.

  ```bash
  kubectl -n "$NS" get pods -l app=web -o wide | tee outputs/62_web_pods_after_drain.txt
  ```
  ```bash
  kubectl -n "$NS" describe pdb web-pdb | sed -n '1,220p' | tee outputs/63_describe_pdb_after_drain.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa eventos del namespace (si hubo bloqueos/evictions).

  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 50 | tee outputs/64_events_tail_after_drain.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Obligatorio) Devuelve el nodo al scheduling normal.

  ```bash
  kubectl uncordon "$NODE_NAME" | tee outputs/65_uncordon_node.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el tester siguió en 200 (captura últimos 30 logs).

  ```bash
  kubectl -n "$NS" logs pod/tester --tail=30 | tee outputs/66_tester_tail_after_drain.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Validación final.

Ejecutarás un checklist final para dejar evidencia del estado: Deployment/Service/Pods/endpoints/PDB/eventos.

#### Tarea 7.1

- {% include step_label.html %} regresa a la **primera terminal** y ejecuta el checklist y guarda evidencia.

  > **NOTA:** Rompe el proceso de la prueba con **`CTRL+C`**
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$NS" get deploy,svc,pods -o wide | tee outputs/70_final_get_resources.txt
  ```
  ```bash
  kubectl -n "$NS" get endpoints "$APP_NAME" -o wide | tee outputs/71_final_endpoints.txt
  ```
  ```bash
  kubectl -n "$NS" get pdb -o wide | tee outputs/72_final_pdb.txt
  ```
  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 40 | tee outputs/73_final_events_tail.txt
  ```

- {% include step_label.html %} (Recomendado) Captura `describe` del Deployment final (para verificar strategy + constraints + probes).

  ```bash
  kubectl -n "$NS" describe deploy "$APP_NAME" | sed -n '1,260p' | tee outputs/74_final_describe_deploy.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Limpieza del laboratorio

Eliminarás el namespace del laboratorio. Si creaste un clúster solo para esta práctica, también lo eliminarás.

#### Tarea 8.1

- {% include step_label.html %} Elimina el namespace del laboratorio (incluye app + tester + PDB).

  ```bash
  kubectl delete ns "$NS" | tee outputs/80_delete_ns.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el namespace ya no existe.

  ```bash
  kubectl get ns | grep "$NS" || echo "OK: namespace eliminado" | tee outputs/81_verify_ns_deleted.txt
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
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}