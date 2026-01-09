---
layout: lab
title: "Práctica 6: Simulación de fallos con Chaos Mesh en EKS"
permalink: /lab6/lab6/
images_base: /labs/lab6/img
duration: "60 minutos"
objective:
  - "Instalar Chaos Mesh en un clúster EKS, desplegar una app de prueba y ejecutar experimentos controlados (Pod kill, latencia de red y stress de CPU/memoria) para observar el comportamiento real de Kubernetes (restarts, endpoints, disponibilidad y recuperación), validando con kubectl (events/logs/describe) y buenas prácticas de blast radius (scoping por labels/namespaces)."
prerequisites:
  - "Amazon EKS accesible con kubectl."
  - "Windows + Visual Studio Code + Terminal GitBash."
  - "Herramientas: AWS CLI v2, kubectl, eksctl, Helm 3."
  - "Docker Desktop (opcional, para smoke test local)."
  - "Permisos Kubernetes: crear namespaces, deployments, services y recursos en el namespace de Chaos Mesh."
  - "Permisos AWS: acceso para consultar EKS"
introduction:
  - "Chaos Mesh es una plataforma cloud-native de chaos engineering basada en CRDs: declaras un experimento (PodChaos, NetworkChaos, StressChaos, etc.) y el controlador lo ejecuta en el clúster. En esta práctica no solo aplicarás YAML: medirás antes/durante/después con kubectl (Ready/NotReady, endpoints, restarts, eventos, latencia percibida) y limitarás el impacto con namespaces, labels, mode y duration."
slug: lab6
lab_number: 6
final_result: >
  Al finalizar tendrás Chaos Mesh instalado en EKS y habrás ejecutado fallos controlados (pod-kill, delay de red y stress de CPU/mem), observando evidencia clara de recuperación (pods reemplazados), degradación (latencia) y resiliencia (Service/endpoints), con un enfoque de blast radius usando selectors por namespace/labels y duración acotada.
notes:
  - "Recomendación fuerte: ejecuta en un clúster de laboratorio o dev/stage. NO producción."
  - "Usa mode: one/all con intención y siempre define duration cuando aplique (NetworkChaos/StressChaos)."
  - "Mide baseline → durante → después. Si no, el caos no enseña nada."
  - "CKAD: refuerza skills de labels/selectors, Deployments/Services, endpoints, y troubleshooting con kubectl get/describe/events/logs."
references:
  - text: "Chaos Mesh - Production installation using Helm"
    url: https://chaos-mesh.org/docs/production-installation-using-helm/
  - text: "Chaos Mesh - Uninstall using Helm"
    url: https://chaos-mesh.org/docs/uninstallation/
  - text: "Chaos Mesh - Define the scope of chaos experiments (selectors)"
    url: https://chaos-mesh.org/docs/define-chaos-experiment-scope/
  - text: "Chaos Mesh - Simulate Pod faults (PodChaos)"
    url: https://chaos-mesh.org/docs/simulate-pod-chaos-on-kubernetes/
  - text: "Chaos Mesh - Simulate Network faults (NetworkChaos)"
    url: https://chaos-mesh.org/docs/simulate-network-chaos-on-kubernetes/
  - text: "Chaos Mesh - Simulate Stress scenarios (StressChaos)"
    url: https://chaos-mesh.org/docs/simulate-heavy-stress-on-kubernetes/
  - text: "AWS - Amazon EKS Pricing"
    url: https://aws.amazon.com/eks/pricing/
prev: /lab5/lab5/
next: /lab7/lab7/
---

---

### Tarea 1. Preparar la carpeta de la práctica y validar contexto

Confirmarás tu cuenta/región de AWS, asegurarás que `kubectl` apunta al clúster correcto y dejarás variables listas para ejecutar los experimentos con un blast radius controlado.

> **IMPORTANTE:** Chaos Mesh ejecuta acciones con alto privilegio (daemon en nodos). Úsalo **solo** en un clúster de laboratorio/dev/stage.
{: .lab-note .important .compact}

> **NOTA (CKAD):** Practicarás troubleshooting con `kubectl describe`, `kubectl get events`, `kubectl logs`, `-w` (watch), endpoints y verificación de rollouts.
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo asignado al curso con el usuario que tenga permisos administrativos.

- {% include step_label.html %} Abre el **`Visual Studio Code`**. Lo puedes encontrar en el **Escritorio** del ambiente o puedes buscarlo en las aplicaciones de Windows.

- {% include step_label.html %} Una vez abierto **VSCode**, da clic en el icono de la imagen para abrir la terminal, se encuentra en la parte superior derecha.

  {% include step_image.html %}

- {% include step_label.html %} Usa la terminal de **`Git Bash`**, da clic como lo muestra la imagen.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar dentro de la carpeta raíz del curso (por ejemplo `labs-eks-ckad`).

  > **Nota:** Si vienes de otra práctica, usa `cd ..` hasta volver a la raíz.
  {: .lab-note .info .compact}

- {% include step_label.html %} Crea el directorio de trabajo de la práctica y entra a él.

  ```bash
  mkdir lab06 && cd lab06
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea la estructura de carpetas del laboratorio.

  ```bash
  mkdir -p k8s/app k8s/chaos scripts outputs
  ```

- {% include step_label.html %} Confirma la estructura de directorios.

  ```bash
  find . -maxdepth 3 -type d | sort
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida la identidad de AWS y región (reduce el riesgo de operar en otra cuenta/región).

  ```bash
  aws sts get-caller-identity
  ```
  ```bash
  aws configure get region || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define la variable base para el namespace de la practica.

  ```bash
  export APP_NS="chaos-demo"
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. (Opcional) Crear un clúster EKS de laboratorio para Chaos Mesh

Crearás un clúster EKS con Managed Node Group usando `eksctl`, ideal si no tienes uno disponible o si quieres aislar el laboratorio.

> **IMPORTANTE:** Si ya tienes un clúster EKS y `kubectl` se conecta correctamente, **omite** esta tarea.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} (Recomendado) Revisa versiones soportadas en la región antes de elegir `K8S_VERSION`.

  ```bash
  export AWS_REGION="us-west-2"
  ```
  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de que la versión elegida aparece en la lista, usaremos la version **`1.33`**.

- {% include step_label.html %} Define variables del clúster (puedes ajustar nombres/región/versión).

  ```bash
  export AWS_REGION="us-west-2"
  export CLUSTER_NAME="eks-chaosmesh-lab"
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

- {% include step_label.html %} Crea el clúster con Managed Node Group (2 nodos) usando `eksctl`.

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

- {% include step_label.html %} Configura kubeconfig y el contexto del cluster.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  kubectl cluster-info
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

- {% include step_label.html %} Verifica que el cluster y el node group este correctamente creado, con los siguientes comandos.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.status" --output text
  ```
  ```bash
  aws eks describe-nodegroup --cluster-name "$CLUSTER_NAME" --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION" --query "nodegroup.status" --output text
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

### Tarea 3. Desplegar app objetivo + cliente de prueba (curl)

Desplegarás una app simple (`whoami`) con 2 réplicas y un `Service` para tener un objetivo claro; además, crearás un Pod `curl` para medir impacto desde dentro del clúster (latencia/errores) sin variables externas.

#### Tarea 3.1

- {% include step_label.html %} Realiza el Smoke test local de la imagen (reduce troubleshooting innecesario).

  ```bash
  docker run --rm -p 8080:80 traefik/whoami:v1.10.3
  ```
  {% include step_image.html %}

- {% include step_label.html %} **Abre otra terminal Bash** y ejecuta el siguiente comando:

  ```bash
  curl -s http://localhost:8080 | head
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el namespace de la app (blast radius controlado).

  > **IMPORTANTE:** El comando se ejecuta en la **primera terminal bash** ejecuta **`CTRL+C`**
  {: .lab-note .important .compact}

  ```bash
  kubectl create ns "$APP_NS"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el manifiesto de app + service con labels claros.

  ```bash
  cat > k8s/app/01-whoami.yaml <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: whoami
    namespace: ${APP_NS}
    labels:
      app: whoami
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: whoami
    template:
      metadata:
        labels:
          app: whoami
      spec:
        containers:
        - name: whoami
          image: traefik/whoami:v1.10.3
          ports:
          - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: whoami
    namespace: ${APP_NS}
  spec:
    selector:
      app: whoami
    ports:
    - name: http
      port: 80
      targetPort: 80
  EOF
  ```

- {% include step_label.html %} Aplica el manifiesto y espera el rollout.

  ```bash
  kubectl apply -f k8s/app/01-whoami.yaml
  kubectl -n "$APP_NS" rollout status deploy/whoami
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea un Pod `curl` para pruebas internas y espera a que esté Ready.

  ```bash
  kubectl -n "$APP_NS" run curl --image=curlimages/curl:8.10.1 --restart=Never -- sleep 3600
  ```
  ```bash
  kubectl -n "$APP_NS" wait --for=condition=Ready pod/curl --timeout=120s
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica los objetos creados.

  ```bash
  kubectl -n "$APP_NS" get deploy,svc,pods -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica los endpoints.

  ```bash
  kubectl -n "$APP_NS" get endpoints whoami -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Realiza la prueba del endpoint.

  ```bash
  kubectl -n "$APP_NS" exec -it curl -- sh -lc 'for i in $(seq 1 4); do curl -s http://whoami/; done'
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

### Tarea 4. Instalar Chaos Mesh con Helm y verificar componentes

Instalarás Chaos Mesh (CRDs + control plane + daemon) usando Helm, validarás que todos los pods estén Running y, opcionalmente, accederás al dashboard por port-forward.

#### Tarea 4.1

- {% include step_label.html %} Confirma el runtime de los nodos (esperas containerd).

  ```bash
  kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{end}'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que Helm 3 está disponible.

  ```bash
  helm version --short
  ```
  {% include step_image.html %}

- {% include step_label.html %} Agrega el repo Helm oficial.

  ```bash
  helm repo add chaos-mesh https://charts.chaos-mesh.org
  helm repo update
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora instala Chaos Mesh para containerd (EKS).

  ```bash
  export CHAOS_NS="chaos-mesh"

  MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL="*" \
  helm upgrade --install chaos-mesh chaos-mesh/chaos-mesh \
    -n "$CHAOS_NS" --create-namespace \
    --set-string chaosDaemon.runtime=containerd \
    --set-string chaosDaemon.socketPath=/run/containerd/containerd.sock
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que existan CRDs de Chaos Mesh.

  ```bash
  kubectl get crds | grep chaos-mesh || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que los componentes estén listos (pods/deploy).

  ```bash
  kubectl -n "$CHAOS_NS" get pods -o wide
  ```
  ```bash
  kubectl -n "$CHAOS_NS" get deploy
  ```
  ```bash
  kubectl -n "$CHAOS_NS" get ds chaos-daemon
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Accede al dashboard por port-forward. Ejecuta el comando en la **primera terminal bash**

  > **NOTA:** El dashboard escucha en el puerto 2333. Mantén RBAC/seguridad habilitada por defecto.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$CHAOS_NS" get svc | grep chaos-dashboard || true
  kubectl port-forward -n "$CHAOS_NS" svc/chaos-dashboard 2333:2333
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre el navegador Chrome/Edge y pega la siguiente URL.

  > **IMPORTANTE:** Por el momento la interfaz grafica no sera usada en la practica.
  {: .lab-note .important .compact}

  ```bash
  http://localhost:2333
  ```
  {% include step_image.html %}

- {% include step_label.html %} **(Opcional SOLO LAB)** Si necesitas un dashboard “sin fricción”, puedes reinstalar con `dashboard.securityMode=false`.

  > **IMPORTANTE:** Úsalo solo en laboratorio.
  {: .lab-note .important .compact}

  ```bash
  # SOLO SI LO NECESITAS (lab):
  # helm uninstall chaos-mesh -n "$CHAOS_NS"
  # helm install chaos-mesh chaos-mesh/chaos-mesh \
  #   --namespace "$CHAOS_NS" --create-namespace \
  #   --set dashboard.securityMode=false
  ```

- {% include step_label.html %} Regresa a la **primera terminal** y corta el **port-forward** con **`CTRL+C`**, ejecuta los siguientes comandos. Si algo falla, revisa eventos y logs del controlador.

  > **NOTA:** Si no hay errores avanza a la siguiente tarea.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$CHAOS_NS" get events --sort-by=.lastTimestamp | tail -n 30
  ```
  ```bash
  kubectl -n "$CHAOS_NS" logs deploy/chaos-controller-manager --tail=80 || true
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Experimento 1 — PodChaos (pod-kill) y observación de auto-recuperación

Ejecutarás un `PodChaos` que mate un pod (mode: one) de la app, observarás el reemplazo automático por el Deployment, y validarás continuidad del servicio con el pod restante.

#### Tarea 5.1

- {% include step_label.html %} Crea el manifiesto de PodChaos (selector por namespace + label = blast radius).

  ```bash
  cat > k8s/chaos/01-pod-kill.yaml <<EOF
  apiVersion: chaos-mesh.org/v1alpha1
  kind: PodChaos
  metadata:
    name: whoami-pod-kill-one
    namespace: ${CHAOS_NS}
  spec:
    action: pod-kill
    mode: one
    selector:
      namespaces:
        - ${APP_NS}
      labelSelectors:
        app: whoami
  EOF
  ```

- {% include step_label.html %} Aplica el experimento.

  ```bash
  kubectl apply -f k8s/chaos/01-pod-kill.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa el reemplazo del pod (watch).

  ```bash
  kubectl -n "$APP_NS" get pods -l app=whoami -w
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica el estado del experimento y continuidad del servicio.

  ```bash
  kubectl -n "$CHAOS_NS" get podchaos whoami-pod-kill-one -o wide
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$APP_NS" exec -it curl -- sh -lc 'for i in $(seq 1 5); do curl -s -o /dev/null -w "time=%{time_total}\n" http://whoami/; done' | tee outputs/baseline_latency.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$APP_NS" get events --sort-by=.lastTimestamp | tail -n 25
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

---

### Tarea 6. Experimento 2 — NetworkChaos (delay) y validación de latencia percibida

Inyectarás latencia (ej. 200ms) al tráfico hacia los pods `whoami` por 60s y medirás la diferencia con `curl` (tiempos de respuesta), validando antes/durante/después.

#### Tarea 6.1

- {% include step_label.html %} Mide baseline (antes del caos).

  ```bash
  kubectl -n "$APP_NS" exec -it curl -- sh -lc 'for i in $(seq 1 5); do curl -s -o /dev/null -w "time=%{time_total}\n" http://whoami/; done' | tee outputs/baseline_latency.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el experimento NetworkChaos con duration (impacto acotado).

  ```bash
  cat > k8s/chaos/02-network-delay.yaml <<EOF
  apiVersion: chaos-mesh.org/v1alpha1
  kind: NetworkChaos
  metadata:
    name: whoami-network-delay
    namespace: ${CHAOS_NS}
  spec:
    action: delay
    mode: all
    duration: "60s"
    selector:
      namespaces:
        - ${APP_NS}
      labelSelectors:
        app: whoami
    delay:
      latency: "200ms"
      correlation: "100"
      jitter: "0ms"
  EOF
  ```

- {% include step_label.html %} Aplica el experimento y mide durante el caos.

  ```bash
  kubectl apply -f k8s/chaos/02-network-delay.yaml
  ```
  ```bash
  kubectl -n "$APP_NS" exec -it curl -- sh -lc 'for i in $(seq 1 5); do curl -s -o /dev/null -w "time=%{time_total}\n" http://whoami/; done' | tee outputs/during_delay_latency.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera a que termine y mide después del caos.

  ```bash
  sleep 65
  kubectl -n "$APP_NS" exec -it curl -- sh -lc 'for i in $(seq 1 5); do curl -s -o /dev/null -w "time=%{time_total}\n" http://whoami/; done' | tee outputs/after_delay_latency.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora valida la inyección correcta del chaos.

  ```bash
  kubectl -n "$APP_NS" get podnetworkchaos -o wide || true
  ```
  ```bash
  kubectl -n "$APP_NS" get events --sort-by=.lastTimestamp | tail -n 25
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r6 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r6 %}

---

### Tarea 7. Experimento 3 — StressChaos (CPU o memoria) y observación de degradación

Inyectarás stress dentro del contenedor (CPU o memoria) para observar degradación (latencia mayor, posible throttling) sin necesariamente tumbar el servicio; esto ayuda a razonar sobre requests/limits y SLOs.

#### Tarea 7.1

- {%include step_label.html %} Verifica que kubectl top funciona.

  ```bash
  kubectl top nodes >/dev/null 2>&1 && echo "OK: metrics-server disponible" || echo "WARN: kubectl top no disponible (metrics-server faltante)"
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Recomendado) Mide baseline de latencia (si no lo hiciste antes).

  ```bash
  kubectl -n "$APP_NS" exec -it curl -- sh -lc \
  'for i in $(seq 1 8); do curl -s -o /dev/null -w "time=%{time_total}\n" http://whoami/; done' \
  | tee outputs/baseline_for_stress.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea un StressChaos de CPU (30s, mode: one).

  ```bash
  cat > k8s/chaos/03-cpu-stress.yaml <<EOF
  apiVersion: chaos-mesh.org/v1alpha1
  kind: StressChaos
  metadata:
    name: whoami-cpu-stress
    namespace: ${CHAOS_NS}
  spec:
    mode: one
    duration: "30s"
    selector:
      namespaces:
        - ${APP_NS}
      labelSelectors:
        app: whoami
    stressors:
      cpu:
        workers: 1
        load: 80
  EOF
  ```

- {% include step_label.html %} Aplica el stress.

  ```bash
  kubectl apply -f k8s/chaos/03-cpu-stress.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica estado del experimento (si está “Injected” / running).

  ```bash
  kubectl -n "$CHAOS_NS" get stresschaos whoami-cpu-stress -o wide
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$CHAOS_NS" describe stresschaos whoami-cpu-stress | sed -n '1,220p'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Mide el stress durante el caos.

  ```bash
  kubectl -n "$APP_NS" exec -it curl -- sh -lc \
  'for i in $(seq 1 15); do curl -s -o /dev/null -w "time=%{time_total}\n" http://whoami/; sleep 1; done' \
  | tee outputs/during_cpu_stress_latency.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica estado del experimento y del workload (pods siguen presentes).

  > **NOTA:** Es normal observar latencia mayor sin que el pod deje de estar Ready. “Ready” no significa “cumple tu SLO”.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$CHAOS_NS" get stresschaos whoami-cpu-stress -o wide
  ```
  ```bash
  kubectl -n "$APP_NS" get pods -l app=whoami -o wide
  ```
  ```bash
  kubectl -n "$APP_NS" get events --sort-by=.lastTimestamp | tail -n 20
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r7 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r7 %}

---

### Tarea 8. Validación final integral (antes de limpiar)

Harás un checklist final para confirmar que la app sigue respondiendo, que no hay experimentos activos “olvidados” y que tu evidencia (outputs) quedó guardada.

#### Tarea 8.1

- {% include step_label.html %} Verifica salud del servicio y balanceo simple.

  ```bash
  kubectl -n "$APP_NS" exec -it curl -- sh -lc 'for i in $(seq 1 8); do curl -s http://whoami/; done'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa endpoints y estado de pods.

  ```bash
  kubectl -n "$APP_NS" get pods -l app=whoami
  ```
  ```bash
  kubectl -n "$APP_NS" get endpoints whoami -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa CRs de chaos (deberían estar Completed o ya sin efecto por duration).

  ```bash
  kubectl -n "$CHAOS_NS" get podchaos,networkchaos,stresschaos -o wide || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda un snapshot final de eventos del namespace de la app.

  ```bash
  kubectl -n "$APP_NS" get events --sort-by=.lastTimestamp | tail -n 40 | tee outputs/final_app_events_tail.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r8 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r8 %}

---

### Tarea 9. Limpieza del laboratorio

Eliminarás los experimentos, el namespace de la app y, si Chaos Mesh fue solo para esta práctica, lo desinstalarás para dejar el clúster limpio.

#### Tarea 9.1

- {% include step_label.html %} Elimina los experimentos (CRDs) aplicados.

  ```bash
  kubectl delete -f k8s/chaos/01-pod-kill.yaml --ignore-not-found
  ```
  ```bash
  kubectl delete -f k8s/chaos/02-network-delay.yaml --ignore-not-found
  ```
  ```bash
  kubectl delete -f k8s/chaos/03-cpu-stress.yaml --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el namespace de la app.

  ```bash
  kubectl delete ns "$APP_NS"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Desinstala Chaos Mesh si fue solo para el laboratorio.

  ```bash
  helm uninstall chaos-mesh -n "$CHAOS_NS" || true
  ```
  {% include step_image.html %}
  ```bash
  kubectl delete ns "$CHAOS_NS" || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que ya no existan los namespaces de la práctica.

  ```bash
  kubectl get ns | egrep "chaos-demo|chaos-mesh" || echo "OK: namespaces eliminados"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el clúster creado por `eksctl`.

  > **NOTA:** El cluster tardara **7 minutos** aproximadamente en eliminarse.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el cluster se haya eliminado correctamente.

  > **NOTA:** Costos principales:
  - Chaos Mesh: software open-source (sin costo de licencia).
  - EKS: costo por clúster/hora + costo de nodos (EC2/Fargate) que ya tengas corriendo.
  - Esta práctica **no crea** ALB/NLB ni servicios AWS adicionales por sí misma 
  {: .lab-note .info .compact}

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || echo "Cluster eliminado"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r9 %}{{ results[8] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r9 %}