---
layout: lab
title: "Práctica 5: Configuración de probes en Pods y validación de comportamiento"
permalink: /lab5/lab5/
images_base: /labs/lab5/img
duration: "50 minutos"
objective:
  - "Diseñar, implementar y ajustar livenessProbe, readinessProbe y startupProbe en un Deployment en EKS para observar y validar su comportamiento real: reinicios por liveness, exclusión de endpoints por readiness y protección de arranque lento con startup; todo con troubleshooting tipo CKAD usando kubectl describe, events, logs, rollout y pruebas de tráfico desde dentro del clúster."
prerequisites:
  - "Amazon EKS accesible con kubectl"
  - "Windows + Visual Studio Code + Terminal GitBash"
  - "Herramientas: AWS CLI v2, kubectl, eksctl"
  - "Docker Desktop (Docker local)"
  - "(Recomendado) git"
  - "Permisos Kubernetes: crear Namespace, Deployment, Service y Pods de debug"
  - "Permisos AWS: ECR (crear repo, login, push)"
introduction:
  - "Las probes son el “contrato de salud” entre tu aplicación y Kubernetes: liveness decide cuándo reiniciar un contenedor si está vivo pero atorado; readiness decide si un Pod recibe tráfico (si falla, se quita de los endpoints del Service); startup es un escudo para arranques lentos (si está configurada, deshabilita liveness/readiness hasta que el contenedor arranque correctamente). En este laboratorio provocarás comportamientos reales (reinicios, exclusión de endpoints, arranque lento protegido) y los validarás con troubleshooting de estilo CKAD."
slug: lab5
lab_number: 5
final_result: |
  Al finalizar habrás construido una app demo y validado en vivo los 3 comportamientos clave de Kubernetes:
  - Un liveness mal calibrada puede reiniciar contenedores de forma prematura (especialmente en arranques lentos).
  - Un startupProbe protege el arranque y evita reinicios prematuros.
  - Un readinessProbe controla el tráfico quitando Pods de los endpoints del Service sin reiniciarlos; todo respaldado con evidencia en describe/events/logs/rollout y pruebas de tráfico desde dentro del clúster.
notes:
  - "CKAD: práctica núcleo (probes + troubleshooting con kubectl describe/events/logs/rollout + pods temporales para pruebas)."
  - "Este laboratorio usa ECR para tener una imagen reproducible. Si tu clúster no tiene salida a internet o quieres evitar imágenes públicas, ECR es el camino natural en AWS."
  - "Regla de oro: readiness = “¿puedo recibir tráfico?”; liveness = “¿debo reiniciarte?”; startup = “todavía estoy arrancando; no me mates”."
references:
  - text: "Kubernetes - Container probes (liveness/readiness/startup)"
    url: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
  - text: "Kubernetes - Configure liveness, readiness and startup probes"
    url: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
  - text: "Kubernetes - Debug Pods (events/describe/logs)"
    url: https://kubernetes.io/docs/tasks/debug/debug-application/
  - text: "AWS - Amazon ECR (conceptos y uso)"
    url: https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html
  - text: "AWS - Auth a ECR (get-login-password)"
    url: https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html
prev: /lab4/lab4/
next: /lab6/lab6/
---

---

### Tarea 1. Preparar la carpeta de la práctica y el entorno

Crearás la carpeta del laboratorio, abrirás **GitBash** dentro de **VS Code**, definirás variables y validarás que `aws` y `kubectl` apuntan a la cuenta y clúster correctos.

> **NOTA (CKAD):** Practicarás diagnóstico de probes con `kubectl describe`, `kubectl get events`, `kubectl logs`, `kubectl rollout status`, JSONPath y un Pod temporal para pruebas desde dentro del clúster.
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo asignado al curso con el usuario que tenga permisos administrativos.

- {% include step_label.html %} Abre el **`Visual Studio Code`** (desde el Escritorio o el menú de Windows).

- {% include step_label.html %} Abre la terminal integrada en VS Code (ícono de terminal en la parte superior derecha).

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **`Git Bash`** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar en la carpeta raíz de tu repositorio/laboratorios (por ejemplo **`labs-eks-ckad`**).

  > **Nota:** Si estás dentro del directorio de otra práctica, usa `cd ..` para volver a la raíz.
  {: .lab-note .info .compact}

- {% include step_label.html %} Crea el directorio de trabajo del laboratorio y entra en él.

  ```bash
  mkdir lab05 && cd lab05
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea la estructura de carpetas estándar del laboratorio.

  ```bash
  mkdir -p app k8s scripts outputs
  mkdir -p k8s/bad k8s/fixed
  ```

- {% include step_label.html %} Confirma la estructura de directorios creada.

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

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear un clúster EKS para la práctica (opcional)

Crearás un clúster EKS con Managed Node Group usando `eksctl`. **Omite esta tarea** si ya tienes un clúster y `kubectl` se conecta correctamente.

> **IMPORTANTE:** Crear un clúster puede tardar varios minutos y genera costos de EKS + EC2. Elimina el clúster al final si es solo laboratorio.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} (Recomendado) Revisa versiones soportadas en tu región antes de elegir `K8S_VERSION`.

  ```bash
  export AWS_REGION="us-west-2"
  ```
  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de que la versión elegida aparece en la lista, usaremos la version **`1.33`**.

- {% include step_label.html %} Define variables del clúster (si ya definiste `AWS_REGION`, reutilízala).

  ```bash
  export CLUSTER_NAME="eks-probes-lab"
  export NODEGROUP_NAME="mng-1"
  export K8S_VERSION="1.33"
  ```

- {% include step_label.html %} Valida que las variables queden bien.

  ```bash
  echo "AWS_REGION=$AWS_REGION"
  echo "CLUSTER_NAME=$CLUSTER_NAME"
  echo "NODEGROUP_NAME=$NODEGROUP_NAME"
  echo "K8S_VERSION=$K8S_VERSION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el clúster EKS con 2 nodos administrados.

  > **NOTA:** `eksctl` puede crear VPC/subnets/SG/roles además del clúster y el Managed Node Group.
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

- {% include step_label.html %} Configura `kubeconfig` y valida el estado del clúster.

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  kubectl cluster-info
  ```
  ```bash
  kubectl get nodes -o wide
  ```
  ```bash
  kubectl get ns
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica el estado del clúster y nodegroup desde AWS.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.status" --output text
  ```
  ```bash
  aws eks describe-nodegroup --cluster-name "$CLUSTER_NAME" --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION" --query "nodegroup.status" --output text
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear la app demo (Docker) y publicarla en ECR

Construirás una imagen con una app HTTP sencilla (endpoints `/healthz` y `/readyz`) que permite simular **arranque lento** y **degradación de readiness** para observar el efecto real de las probes.

#### Tarea 3.1

- {% include step_label.html %} Crea el archivo `app/server.py`.

  > **Nota:** `/readyz` representa “¿puedo recibir tráfico?” y `/healthz` “¿debo reiniciar este contenedor?”. Este es el patrón recomendado para separar readiness vs liveness.
  {: .lab-note .info .compact}

  ```bash
  cat > app/server.py <<'EOF'
  import os, time, threading
  from flask import Flask

  app = Flask(__name__)
  START = time.time()
  STARTUP_DELAY = int(os.getenv("STARTUP_DELAY", "0"))
  FAIL_AFTER = int(os.getenv("FAIL_AFTER", "0"))  # 0 = nunca falla healthz
  READY_FILE = "/tmp/ready"

  # Simula arranque lento: crea READY_FILE hasta después de STARTUP_DELAY
  def init():
    time.sleep(STARTUP_DELAY)
    with open(READY_FILE, "w") as f:
      f.write("ready")

  threading.Thread(target=init, daemon=True).start()

  @app.get("/")
  def index():
    return f"pod={os.getenv('HOSTNAME','unknown')} uptime={int(time.time()-START)}s\n", 200

  @app.get("/readyz")
  def readyz():
    try:
      open(READY_FILE, "r").close()
      return "ready\n", 200
    except:
      return "not-ready\n", 503

  @app.get("/healthz")
  def healthz():
    if FAIL_AFTER > 0 and (time.time() - START) > FAIL_AFTER:
      return "unhealthy\n", 500
    return "ok\n", 200

  if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
  EOF
  ```

- {% include step_label.html %} Crea el archivo `app/requirements.txt`.

  ```bash
  cat > app/requirements.txt <<'EOF'
  flask==3.0.3
  EOF
  ```

- {% include step_label.html %} Crea el archivo `app/Dockerfile`.

  ```bash
  cat > app/Dockerfile <<'EOF'
  FROM python:3.12-slim
  WORKDIR /app
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt
  COPY server.py .
  EXPOSE 8080
  CMD ["python","server.py"]
  EOF
  ```

- {% include step_label.html %} (Opcional) Crea `.dockerignore` para builds más rápidos.

  > **Nota:** Buena practica
  {: .lab-note .info .compact}

  ```bash
  cat > app/.dockerignore <<'EOF'
  __pycache__/
  *.pyc
  .pytest_cache/
  .venv/
  EOF
  ```

- {% include step_label.html %} Verifica la creación de los archivos en el directorio **app/**.

  ```bash
  ls -la app/
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables ECR y la imagen a publicar.

  ```bash
  export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  export ECR_REPO="probes-repo"
  export ECR_URI="$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO"
  export IMAGE_TAG="v1"
  export IMAGE="$ECR_URI:$IMAGE_TAG"
  ```
  ```bash
  echo "IMAGE=$IMAGE"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el repositorio ECR si no existe.

  ```bash
  aws ecr describe-repositories --repository-names "$ECR_REPO" --region "$AWS_REGION" >/dev/null 2>&1 || aws ecr create-repository --repository-name "$ECR_REPO" --region "$AWS_REGION" --output table
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inicia sesión en ECR, construye y publica la imagen.

  ```bash
  aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Construye la imagen Docker.

  ```bash
  docker build -t "$IMAGE" app
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora publica la imagen al repositorio.

  ```bash
  docker push "$IMAGE"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda evidencia del repositorio creado.

  ```bash
  aws ecr describe-repositories --repository-names "$ECR_REPO" --region "$AWS_REGION" --output table | tee outputs/ecr_repo.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que se haya guardado correctamente.

  ```bash
  cat outputs/ecr_repo.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Escenario A (anti‑patrón): liveness mata un arranque lento

Desplegarás la app con **arranque lento** y una **livenessProbe mal diseñada** (usa `/readyz` como si fuera liveness). Como `/readyz` regresa 503 durante el arranque, Kubernetes reiniciará el contenedor repetidamente.

> **IMPORTANTE:** Este anti‑patrón es común: usar readiness como liveness. Liveness es para “proceso atascado”, no para “todavía no estoy listo”.
{: .lab-note .important .compact}

#### Tarea 4.1

- {% include step_label.html %} Crea el namespace del laboratorio.

  ```bash
  export NS="probes-lab"
  kubectl create ns "$NS"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el manifiesto "malo" (sin startupProbe y liveness agresiva apuntando a `/readyz`).

  ```bash
  cat > k8s/bad/01-deploy-no-startupprobe.yaml <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: probe-demo
    namespace: $NS
    labels:
      app: probe-demo
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: probe-demo
    template:
      metadata:
        labels:
          app: probe-demo
      spec:
        containers:
        - name: app
          image: $IMAGE
          ports:
          - containerPort: 8080
          env:
          - name: STARTUP_DELAY
            value: "40"
          # Anti-patrón: liveness valida "readiness" (/readyz) y mata el arranque lento
          livenessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 2
  EOF
  ```

- {% include step_label.html %} Aplica el manifiesto y observa el comportamiento (reinicios/eventos).

  > **NOTA:** Puedes cortar el proceso con **`CTRL+C`**
  {: .lab-note .info .compact}

  ```bash
  kubectl apply -f k8s/bad/01-deploy-no-startupprobe.yaml
  ```
  ```bash
  kubectl get pods -n "$NS" -w
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura evidencia del reinicio (describe + restartCount + events).

  ```bash
  POD="$(kubectl get pod -n "$NS" -l app=probe-demo -o jsonpath='{.items[0].metadata.name}')"
  ```
  ```bash
  echo "POD=$POD"
  ```
  {% include step_image.html %}
  ```bash
  kubectl describe pod "$POD" -n "$NS" | sed -n '1,240p' | tee outputs/a_describe_pod_bad.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get pod "$POD" -n "$NS" -o jsonpath='{.status.containerStatuses[0].restartCount}'; echo | tee -a outputs/a_restart_count.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get events -n "$NS" --sort-by=.lastTimestamp | tail -n 30 | tee outputs/a_events_tail.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Escenario B (fix): startupProbe protege el arranque lento

Corregirás el anti‑patrón agregando `startupProbe`. Esta probe actúa como "escudo": mientras startup no sea exitosa, Kubernetes **no ejecuta** liveness/readiness.

#### Tarea 5.1

- {% include step_label.html %} Crea el manifiesto “fixed” con `startupProbe` y separa las responsabilidades:

  - `startupProbe` y `readinessProbe` apuntan a `/readyz`
  - `livenessProbe` apunta a `/healthz`

  ```bash
  cat > k8s/fixed/01-deploy-with-startupprobe.yaml <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: probe-demo
    namespace: $NS
    labels:
      app: probe-demo
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: probe-demo
    template:
      metadata:
        labels:
          app: probe-demo
      spec:
        containers:
        - name: app
          image: $IMAGE
          ports:
          - containerPort: 8080
          env:
          - name: STARTUP_DELAY
            value: "40"
          startupProbe:
            httpGet:
              path: /readyz
              port: 8080
            # Ventana total = failureThreshold * periodSeconds = 12 * 5 = 60s
            failureThreshold: 12
            periodSeconds: 5
            timeoutSeconds: 1
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 1
            periodSeconds: 3
            timeoutSeconds: 1
            failureThreshold: 1
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 2
  EOF
  ```

- {% include step_label.html %} Aplica el manifiesto corregido y espera el rollout.

  ```bash
  kubectl apply -f k8s/fixed/01-deploy-with-startupprobe.yaml
  ```
  ```bash
  kubectl rollout status deploy/probe-demo -n "$NS"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que el Pod **llega a Ready** y que los reinicios se estabilizan (idealmente 0 reinicios tras el fix).

  > **IMPORTANTE:** Revisa cada uno de los archivos generados guardados en el directorio **outputs/**
  {: .lab-note .important .compact}

  > **NOTA:** Es normal ver el evento del error 503 ya que las evaluaciones son cortas en tiempo pero, el problema esta corregido.
  {: .lab-note .info .compact}

  ```bash
  POD="$(kubectl get pod -n "$NS" -l app=probe-demo -o jsonpath='{.items[0].metadata.name}')"
  ```
  ```bash
  kubectl get pod "$POD" -n "$NS" -o wide | tee outputs/b_pod_ready.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get pod "$POD" -n "$NS" -o jsonpath='{.status.containerStatuses[0].restartCount}'; echo | tee outputs/b_restart_count.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl describe pod "$POD" -n "$NS" | egrep -n "Startup|Readiness|Liveness|Restart|Events" -A2 -B2 | tee outputs/b_probe_snippets.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Escenario C: readiness controla endpoints y el tráfico del Service

Desplegarás **2 réplicas** con un **Service** y comprobarás que cuando un Pod falla readiness, Kubernetes lo quita de los **endpoints** sin reiniciarlo.

#### Tarea 6.1

- {% include step_label.html %} Crea un manifiesto con 2 réplicas + Service y readiness agresiva.

  ```bash
  cat > k8s/fixed/02-service-and-readiness.yaml <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: probe-demo
    namespace: $NS
    labels:
      app: probe-demo
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: probe-demo
    template:
      metadata:
        labels:
          app: probe-demo
      spec:
        containers:
        - name: app
          image: $IMAGE
          ports:
          - containerPort: 8080
          env:
          - name: STARTUP_DELAY
            value: "5"
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 1
            periodSeconds: 2
            timeoutSeconds: 1
            failureThreshold: 1
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: probe-demo
    namespace: $NS
  spec:
    selector:
      app: probe-demo
    ports:
    - name: http
      port: 80
      targetPort: 8080
  EOF
  ```

- {% include step_label.html %} Aplica el manifiesto.

  ```bash
  kubectl apply -f k8s/fixed/02-service-and-readiness.yaml
  kubectl rollout status deploy/probe-demo -n "$NS"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica el comportamiento de los pods.

  ```bash
  kubectl get pods -n "$NS" -l app=probe-demo -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica endpoints iniciales (deben existir **2** IPs/Pods listos).

  ```bash
  kubectl -n "$NS" get endpoints probe-demo -o wide | tee outputs/c_endpoints_before.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea un Pod temporal de diagnóstico (curl) para probar el Service desde dentro del clúster.

  ```bash
  kubectl -n "$NS" run curl --image=curlimages/curl:8.10.1 --restart=Never -- sleep 3600
  ```
  ```bash
  kubectl -n "$NS" wait --for=condition=Ready pod/curl --timeout=120s
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba el balanceo y guarda la evidencia (la respuesta incluye `pod=`).

  ```bash
  kubectl -n "$NS" exec -it curl -- sh -lc 'for i in $(seq 1 10); do curl -s http://probe-demo/; done' | tee outputs/c_requests_before.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Fuerza un Pod a quedar **NotReady** (simulando degradación de readiness) borrando `/tmp/ready`.

  ```bash
  kubectl -n "$NS" get pods -l app=probe-demo -o name
  ```
  ```bash
  POD1="$(kubectl -n "$NS" get pod -l app=probe-demo -o jsonpath='{.items[0].metadata.name}')"
  echo "POD1=$POD1"
  ```
  ```bash
  kubectl -n "$NS" exec -it "$POD1" -- sh -lc 'rm -f /tmp/ready; ls -l /tmp/ready || true'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el Pod queda Ready=False y que los endpoints bajan a **1**.

  ```bash
  kubectl -n "$NS" get pod "$POD1" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'; echo | tee outputs/c_pod1_ready_status.txt
  ```
  ```bash
  kubectl -n "$NS" get endpoints probe-demo -o wide | tee outputs/c_endpoints_after.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba de nuevo el Service: ahora el tráfico debe salir **solo** del Pod que permanece Ready.

  ```bash
  kubectl -n "$NS" exec -it curl -- sh -lc 'for i in $(seq 1 10); do curl -s http://probe-demo/; done' | tee outputs/c_requests_after.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura evidencia adicional (describe del Pod y eventos del namespace).

  ```bash
  kubectl -n "$NS" describe pod "$POD1" | egrep -n "Readiness|Ready|Events" -A2 -B2 | tee outputs/c_describe_pod1_readiness.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 25 | tee outputs/c_events_tail.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Validación final (checklist CKAD)

Harás un checklist final: estado de Deployment/Pods/Service/endpoints/eventos para confirmar que entendiste y comprobaste cada comportamiento.

#### Tarea 7.1

- {% include step_label.html %} Ejecuta el checklist y guarda evidencia.

  ```bash
  kubectl -n "$NS" get deploy,svc,pods -o wide | tee outputs/final_get_resources.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get endpoints probe-demo -o wide | tee outputs/final_endpoints.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NS" get events --sort-by=.lastTimestamp | tail -n 30 | tee outputs/final_events_tail.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Revisa logs del contenedor para correlacionar arranque/uptime.

  ```bash
  POD_ANY="$(kubectl -n "$NS" get pod -l app=probe-demo -o jsonpath='{.items[0].metadata.name}')"
  kubectl -n "$NS" logs "$POD_ANY" --tail=80 | tee outputs/final_logs_tail.txt
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Limpieza del laboratorio

Eliminarás el namespace del laboratorio y, si creaste el clúster solo para esta práctica, también lo eliminarás.

#### Tarea 8.1

- {% include step_label.html %} Elimina el namespace del laboratorio.

  ```bash
  kubectl delete ns "$NS"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el namespace ya no existe.

  ```bash
  kubectl get ns | grep "$NS" || echo "OK: namespace eliminado"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el repositorio ECR.

  ```bash
  aws ecr delete-repository --repository-name "$ECR_REPO" --region "$AWS_REGION" --force
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el repositorio se haya eliminado correctamente.

  ```bash
  aws ecr describe-repositories --repository-names "$ECR_REPO" --region "$AWS_REGION" >/dev/null 2>&1 || echo "ECR repo eliminado"
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

{% capture r8 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r8 %}
