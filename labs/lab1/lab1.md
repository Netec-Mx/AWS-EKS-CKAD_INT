---
layout: lab
title: "Práctica 1: Actualización de un clúster EKS y nodos administrados"
permalink: /lab1/lab1/
images_base: /labs/lab1/img
duration: "60 minutos"
objective:
  - "Actualizar de forma controlada un clúster **Amazon EKS (control plane)** y su Managed Node Group, asegurando cero o mínima interrupción mediante buenas prácticas de Kubernetes (réplicas, probes, PDB, rollouts) y validaciones operativas antes/durante/después de la actualización."
prerequisites:
  - "Cuenta AWS con permisos para Amazon EKS/EC2/IAM/ECR."
  - "Un clúster EKS existente con Managed Node Group."
  - "Windows + Visual Studio Code + Terminal GitBash."
  - "AWS CLI v2, kubectl, eksctl y Docker Desktop instalados."
introduction: |
  En Amazon EKS, una actualización segura se ejecuta en tres frentes:
  - **1:** El control plane administrado por AWS.
  - **2:** Los nodos del Managed Node Group con drenado y reemplazo gradual.
  - **3:** Los add-ons críticos (vpc-cni, CoreDNS y kube-proxy) para mantener compatibilidad. En esta práctica desplegarás un “canario” con réplicas, probes y PDB para validar disponibilidad mientras actualizas el control plane, el nodegroup.
slug: lab1
lab_number: 1
final_result: >
  Al finalizar, el clúster EKS tendrá el control plane actualizado a la versión objetivo, el Managed Node Group actualizado (AMI/kubelet compatible), add-ons al día y una aplicación canario demostrará que réplicas + probes + PDB permiten sobrevivir a mantenimientos con mínima o nula interrupción.
notes:
  - "**EKS no permite downgrade de versión**; si necesitas **“volver”**, debes crear otro clúster y migrar workloads."
  - "El upgrade del control plane se realiza 1 version a la vez (ej. 1.32 → 1.33)."
  - "Durante los upgrades, asegúrate de contar con IPs disponibles en subnets (AWS recomienda tener IPs libres; revisa tus subnets antes de actualizar)."
  - "Los rolling updates de Managed Node Groups respetan PDB; si tu PDB impide evictions, la actualización puede fallar o bloquearse."
  - "Planifica según el ciclo de vida (standard/extended support) y política de upgrade del clúster."
references:
  - text: "Actualizar un clúster EKS a una nueva versión de Kubernetes (control plane)"
    url: https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html
  - text: "Actualizar un Managed Node Group (rolling update vs force / PDB)"
    url: https://docs.aws.amazon.com/eks/latest/userguide/update-managed-node-group.html
  - text: "Ciclo de vida de versiones Kubernetes en Amazon EKS"
    url: https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html
  - text: "Mejores prácticas para upgrades de clúster en EKS"
    url: https://docs.aws.amazon.com/eks/latest/best-practices/cluster-upgrades.html
  - text: "Actualizar un add-on de Amazon EKS (resolve conflicts preserve/overwrite)"
    url: https://docs.aws.amazon.com/eks/latest/userguide/updating-an-add-on.html
  - text: "Actualizar el add-on VPC CNI en EKS (procedimiento)"
    url: https://docs.aws.amazon.com/eks/latest/userguide/vpc-add-on-update.html
  - text: "Actualizar CoreDNS add-on en EKS (procedimiento)"
    url: https://docs.aws.amazon.com/eks/latest/userguide/coredns-add-on-update.html
  - text: "eksctl: Cluster upgrades (control plane / nodegroups / add-ons)"
    url: https://docs.aws.amazon.com/eks/latest/eksctl/cluster-upgrade.html
  - text: "Kubernetes: Pod Disruption Budgets (concepto oficial)"
    url: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
  - text: "Amazon EKS Pricing (standard vs extended support)"
    url: https://aws.amazon.com/eks/pricing/
  - text: "Amazon ECR Pricing"
    url: https://aws.amazon.com/ecr/pricing/
prev: /
next: /lab2/lab2/
---

---

### Tarea 1. Preparar el entorno de trabajo

En esta tarea dejarás listo el entorno local, confirmarás identidad/región, seleccionarás el clúster y nodegroup objetivo y asegurarás acceso con `kubectl` antes de ejecutar cualquier actualización.

> **IMPORTANTE:** No existe downgrade en EKS. Verifica cuenta/región/cluster antes de ejecutar upgrades.
{: .lab-note .important .compact}

> **NOTA (CKAD):** Esta práctica refuerza los objetos de app (Deployment/Service), probes, rollouts y PDB. Tu diseño de aplicación determina si sobrevives mantenimientos.
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo asignado al curso con el usuario que tenga permisos administrativos. 

- {% include step_label.html %} Abre **Visual Studio Code (VS Code)**. Puedes hacerlo desde el **Escritorio** o buscándolo en las aplicaciones de Windows.

- {% include step_label.html %} En **VS Code**, abre la **Terminal** (icono en la parte superior derecha o con el menú **View → Terminal**).  

  {% include step_image.html %}

- {% include step_label.html %} Selecciona la terminal **Git Bash** como shell activo.

  {% include step_image.html %}

- {% include step_label.html %} Verifica que **Docker** esté instalado.
  
  > **Nota.** Copia y pega el siguiente comando en la terminal. **La versión puede variar.**
  {: .lab-note .info .compact}
  
  ```bash
  docker --version
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea la estructura base del directorio del curso en el **Escritorio** del equipo asignado.

  > **Importante.** Si es necesario, ajusta manualmente las rutas en la terminal para crear la estructura de directorios.
  {: .lab-note .important .compact}

  - Entra al directorio **Desktop**.
  - Crea un directorio llamado `labs-eks-ckad`.
  - Crea el subdirectorio `lab01`.
  - Dentro, crea los directorios `app`, `k8s`, `scripts` y `outputs`.
  - Puedes ejecutar lo siguiente para crearlo todo.

  ```bash
  cd Desktop/
  mkdir labs-eks-ckad
  mkdir labs-eks-ckad/lab01 && cd labs-eks-ckad/lab01
  mkdir app k8s scripts outputs
  ```

- {% include step_label.html %} Verifica la estructura con el siguiente comando.

  ```bash
  ls -la
  ```
  {% include step_image.html %}

#### Tarea 1.2

- {% include step_label.html %} Abre el directorio del proyecto en **VS Code**.

  > **Nota.** Da clic en el icono como se muestra en la imagen.
  {: .lab-note .info .compact}
  {% include step_image.html %}

- {% include step_label.html %} Clic en **Open Folder**.  
  {% include step_image.html %}

- {% include step_label.html %} Navega al directorio **labs-eks-ckad** (en el Escritorio) y da clic en **Select Folder**.

  > **Nota.** Si aparece la ventana emergente, selecciona **Yes, I trust the authors**.
  {: .lab-note .info .compact}
  {% include step_image.html %}

- {% include step_label.html %} Verás cargado tu directorio para comenzar a trabajar.  
  {% include step_image.html %}

#### Tarea 1.3

- {% include step_label.html %} Verifica tu identidad en AWS (Account/Arn) y la configuración efectiva de AWS CLI. Ejecuta los siguientes comandos en la terminal **Bash** si se cerro vuelvela a abrir.

  > **Nota:** La salida debe mostrar el `Account` correcto y un `Arn` esperado.
  {: .lab-note .info .compact}

  ```bash
  aws sts get-caller-identity
  aws configure get region || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida las versiones de las siguientes herramientas instaladas en el area de trabajo:

  - AWS CLI
  - kubectl
  - eksctl
  - Docker.

  > **IMPORTANTE:** Todos los comandos deben responder sin error. Si `docker` falla, inicia Docker Desktop.
  {: .lab-note .important .compact}

  ```bash
  aws --version
  ```
  ```bash
  kubectl version --client --output=yaml
  ```
  ```bash
  eksctl version
  ```
  ```bash
  docker --version
  ```
  {% include step_image.html %}

{% assign results = site.data["task-results"][page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear un clúster EKS en versión menor
Crearás un cluster en una **versión menor** *(BASE_VER)* con un Managed Node Group, de forma que puedas realizar un upgrade real hacia *(TARGET_VER).*

> **IMPORTANTE:** Esta tarea es “previa” porque la provisión puede tardar varios minutos. Si ya tienes clúster, sáltala.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} (Recomendado) Consulta las versiones soportadas por `eksctl` en la región **us-west-2** Verifica que aparezcan **1.32** y **1.33** sino ajusta **BASE_VER/TARGET_VER** en el siguiente paso.

  > **NOTA:** Evitas elegir versiones no soportadas (la disponibilidad varía por región y tiempo).
  {: .lab-note .info .compact}

  ```bash
  export AWS_REGION="us-west-2"
  ```
  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define las variables (nombre de clúster/nodegroup) y selecciona versiones: **BASE_VER (menor) a TARGET_VER (mayor)**. Ejecuta el comando dentro de la terminal de **GitBash**

  ```bash
  # Nombres
  export CLUSTER_NAME="eks-upgrade-lab"
  export NODEGROUP_NAME="mng-1"
  ```
  ```bash
  # Versiones
  export BASE_VER="1.32"
  export TARGET_VER="1.33"
  ```

- {% include step_label.html %} Verifica que las variables esten correctamente configuradas, escribe la siguiente serie de comandos en la terminal.

  ```bash
  echo "Region=$AWS_REGION"
  echo "Cluster=$CLUSTER_NAME" 
  echo "NodeGroup=$NODEGROUP_NAME"
  echo "BASE_VER=$BASE_VER"
  echo "TARGET_VER=$TARGET_VER"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el clúster con Managed Node Group en la versión BASE_VER.

  > **NOTA:** `eksctl` creará la infraestructura (VPC/subnets/SG/roles) y un Managed Node Group listo para la práctica.
  {: .lab-note .info .compact}

  > **IMPORTANTE:** El cluster tardara aproximadamente **14 minutos** en crearse. Espera el proceso
  {: .lab-note .important .compact}

  ```bash
  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$BASE_VER" \
    --managed \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-type t3.medium --nodes 2 --nodes-min 2 --nodes-max 3
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ejecuta el siguiente comando para verificar la instalación del cluster de EKS.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.status" --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora verifica la instalación del cluster del Management Group.

  ```bash
  aws eks describe-nodegroup --cluster-name "$CLUSTER_NAME" --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION" --query "nodegroup.status" --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Recomendado) Verifica las IPs disponibles en las subredes del clúster (reduce fallas por falta de IPs).

  > **NOTA:** Confirma que hay IPs disponibles en todas las subnets.
  {: .lab-note .info .compact}

  ```bash
  SUBNETS="$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query 'cluster.resourcesVpcConfig.subnetIds' --output text)"
  echo "$SUBNETS"
  ```
  ```bash
  aws ec2 describe-subnets --subnet-ids $SUBNETS --region "$AWS_REGION" --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,Available:AvailableIpAddressCount}' --output table
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el motor de Kubernetes este estable y correctamente configurado. Escribe el siguiente comando

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  kubectl config current-context
  ```
  ```bash
  kubectl get nodes -o wide
  kubectl get ns
  ```
  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Guarda la evidencia inicial de la version del cluster de EKS. **Puedes guardarla en un bloc de notas.**

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.version" --output text | tee outputs/cluster_version_before.txt
  ```
  ```bash
  kubectl get nodes -o wide | tee outputs/nodes_before.txt
  ```
  {% include step_image.html %}
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Baseline + “workload canario” para pruebas

Crearás una app mínima (nginx) y los manifiestos de Kubernetes con **Deployment + Service + Probes + PDB**, para observar que durante el upgrade los pods se reubican sin romper disponibilidad.

#### Tarea 3.1

- {% include step_label.html %} Captura una “foto” de versiones actuales (control plane, nodos y add-ons si aplica). Puede que ya la hayas capturado en la tarea anterior sino, recolecta los datos para la comparacion si hace falta alguno.

  > **NOTA:** Recuerda que puedes guardar la evidencia en un bloc de notas.
  {: .lab-note .info .compact}

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.version" --output text
  ```
  ```bash
  kubectl get nodes -o wide
  ```
  ```bash
  eksctl get addons --cluster "$CLUSTER_NAME" --region "$AWS_REGION" | tee outputs/addons_before.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el archivo `app/Dockerfile` y `app/index.html` (contenedor ultra simple).

  > **NOTA:** El comando se ejecuta desde el directorio **lab01**
  {: .lab-note .info .compact}

  ```bash
  cat > app/Dockerfile <<'EOF'
  FROM nginx:alpine
  COPY index.html /usr/share/nginx/html/index.html
  EOF

  cat > app/index.html <<'EOF'
  <h1>EKS Upgrade Canary</h1>
  <p>OK - version 1</p>
  EOF
  ```

- {% include step_label.html %} Verifica que los archivos se hayan creado correctamente escribe el siguiente comando desde la raíz del directorio **lab01**

  ```bash
  ls -la app
  sed -n '1,20p' app/Dockerfile
  sed -n '1,20p' app/index.html
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el archivo manifiesto del canario: Namespace + Deployment (3 replicas) + Service + PDB.

  > **IMPORTANTE:** Si pones `minAvailable: 3` con `replicas: 3`, bloquearás evictions y puedes romper el rolling update del nodegroup.
  {: .lab-note .important .compact}

  ```bash
  cat > k8s/canary.yaml <<'EOF'
  apiVersion: v1
  kind: Namespace
  metadata:
    name: canary-upgrade
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: canary-web
    namespace: canary-upgrade
    labels:
      app: canary-web
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: canary-web
    template:
      metadata:
        labels:
          app: canary-web
      spec:
        terminationGracePeriodSeconds: 20
        containers:
        - name: web
          image: REPLACE_IMAGE
          ports:
          - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
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
    name: canary-web
    namespace: canary-upgrade
  spec:
    selector:
      app: canary-web
    ports:
    - port: 80
      targetPort: 80
  ---
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata:
    name: canary-web-pdb
    namespace: canary-upgrade
  spec:
    minAvailable: 1
    selector:
      matchLabels:
        app: canary-web
  EOF
  ```

- {% include step_label.html %} Valida el YAML con (dry-run) y verifica el schema del PDB.

  > **NOTA:** Recuerda que `dry-run` debe pasar sin errores.
  {: .lab-note .info .compact}

  ```bash
  kubectl apply --dry-run=client -f k8s/canary.yaml
  ```
  ```bash
  kubectl explain poddisruptionbudget.spec.minAvailable
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Build & Push a ECR + Deploy del canario

Construirás la imagen con Docker local, la subirás a Amazon ECR y desplegarás el canario en el clúster para tener una prueba real durante el upgrade.

#### Tarea 4.1

- {% include step_label.html %} Crea el repositorio ECR (si no existe). Ejecuta el siguiente comando en la terminal de GitBash

  ```bash
  export ECR_REPO="eks-upgrade-canary"
  aws ecr describe-repositories --repository-names "$ECR_REPO" --region "$AWS_REGION" >/dev/null 2>&1 ||   aws ecr create-repository --repository-name "$ECR_REPO" --region "$AWS_REGION" --output table
  ```
  {% include step_image.html %}

- {% include step_label.html %} Escribe el siguiente comando para validar la creación correcta del repositorio.

  ```bash
  aws ecr describe-repositories --repository-names "$ECR_REPO" --region "$AWS_REGION" --query "repositories[0].repositoryUri" --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora realiza la autenticación en Amazon ECR.

  ```bash
  export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  export ECR_URI="$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO"
  aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Publica la imagen `v1`.

  ```bash
  export IMAGE_TAG="v1"
  export IMAGE="$ECR_URI:$IMAGE_TAG"
  ```
  ```bash
  docker build -t "$IMAGE" app
  docker push "$IMAGE"
  ```
  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} Ahora corrobora que si se haya cargado correctamente la imagen al repositorio.

  > **NOTA:** El resultado debe de ser el valor del **digest** de la imagen cargada.
  {: .lab-note .info .compact}

  ```bash
  aws ecr describe-images \
    --repository-name "$ECR_REPO" \
    --region "$AWS_REGION" \
    --query "imageDetails[?imageTags!=null && contains(imageTags, 'v1')].imageDigest" \
    --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Despliega el canario sustituyendo `REPLACE_IMAGE` por tu imagen real. El siguiente comando ya sustituye el valor.

  ```bash
  sed "s#REPLACE_IMAGE#$IMAGE#g" k8s/canary.yaml | kubectl apply -f -
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica el despliegue correcto del namespace canary-upgrade.

  ```bash
  kubectl get ns canary-upgrade
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que los pods esten listos.

  ```bash
  kubectl -n canary-upgrade rollout status deploy/canary-web
  ```
  ```bash
  kubectl -n canary-upgrade get pods -l app=canary-web -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica el pbd y el endpoint del canario.

  ```bash
  kubectl -n canary-upgrade get pdb canary-web-pdb -o wide
  ```
  ```bash
  kubectl -n canary-upgrade get endpoints canary-web
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba el tráfico con `port-forward`. Abre una nueva terminal y ejecuta el siguiente comando.

  ```bash
  kubectl -n canary-upgrade port-forward svc/canary-web 8080:80
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora abre una **tercera terminal** y ejecuta el siguiente comando.

  > **NOTA:** Debes ver el HTML con `OK - version 1`.
  {: .lab-note .info .compact}

  ```bash
  curl -sS http://localhost:8080 | head
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Upgrade del Control Plane

Actualizarás la versión de Kubernetes del clúster (control plane) siguiendo la regla de EKS: **1 minor a la vez**, **sin downgrade**, y monitoreando por `update-id` + salud del API.

#### Tarea 5.1

- {% include step_label.html %} Identifica la versión actual del control plane. Este valor lo debes tener guardado en tu bloc de notas, sino vuelvelo a recolectar.

  > **NOTA:** Debe coincidir con tu estado real (por ejemplo, `BASE_VER` si recién lo creaste).
  {: .lab-note .info .compact}

  > **IMPORTANTE:** Vuelve a la **primera terminal** que abriste ya que ahi se encuentran las variables ya exportadas.
  {: .lab-note .important .compact}

  ```bash
  export CURRENT_VER="$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.version" --output text)"
  echo "Current: $CURRENT_VER"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ejecuta el upgrade del control plane a `TARGET_VER` con `eksctl`.

  > **IMPORTANTE:** El cluster puede tardar **8 minutos** para actualizar. **Mientras en la tercera terminal puedes ejecutar los comandos del siguiente paso sustituye el nombre del cluster y la región**
  {: .lab-note .important .compact}

  ```bash
  echo "Target: $TARGET_VER"

  eksctl upgrade cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --version "$TARGET_VER" --approve
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén el `update-id` más reciente y revisa el estado del update. Ejecuta los 2 comandos.

  > **NOTA:** El `status` debe transicionar de `InProgress` a `Successful` (o equivalente).
  {: .lab-note .info .compact}

  ```bash
  export CP_UPDATE_ID="$(aws eks list-updates --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "updateIds[-1]" --output text)"
  echo "CP_UPDATE_ID=$CP_UPDATE_ID"
  ```
  ```bash
  aws eks describe-update --name "$CLUSTER_NAME" --region "$AWS_REGION" --update-id "$CP_UPDATE_ID" --query "update.{status:status,type:type,errors:errors}" --output yaml
  ```
  {% include step_image.html %}
  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} **Durante el upgrade**, valida salud del API y estado del canario. Ejecutalo en cualquier terminal abierta disponible.

  > **NOTA:** `/readyz` responde y los pods del canario permanecen listos.
  {: .lab-note .info .compact}

  ```bash
  kubectl get --raw=/readyz?verbose | head
  kubectl -n canary-upgrade get pods -l app=canary-web -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que la versión del control plane cambió y el update finalizó.

  > **NOTA:** Versión = `TARGET_VER` y status = `Successful`.
  {: .lab-note .info .compact}

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.version" --output text
  ```
  ```bash
  aws eks describe-update --name "$CLUSTER_NAME" --region "$AWS_REGION" --update-id "$CP_UPDATE_ID" --query "update.status" --output text
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Upgrade del Managed Node Group

Actualizarás el **Managed Node Group (rolling update)**, observarás drenado y reemplazo de nodos, y validarás que el canario se reprograma sin perder disponibilidad gracias a réplicas + probes + PDB.

#### Tarea 6.1

- {% include step_label.html %} Inicia el upgrade del nodegroup con AWS CLI y captura el `update-id`.

  ```bash
  export NG_UPDATE_ID="$(aws eks update-nodegroup-version --cluster-name "$CLUSTER_NAME" --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION" --query 'update.id' --output text)"

  echo "NG_UPDATE_ID=$NG_UPDATE_ID"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica el estado de la actualización, copia y pega el siguiente comando.

  ```bash
  aws eks describe-update --name "$CLUSTER_NAME" --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION" --update-id "$NG_UPDATE_ID" --query "update.{status:status,errors:errors}" --output yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa el drenado/reprogramación de los nodos, pods del canario y PDB.

  > **NOTA:** `Allowed disruptions` debería permitir al menos 1 (si es 0, revisa PDB/replicas).
  {: .lab-note .info .compact}

  ```bash
  kubectl get nodes -o wide
  ```
  ```bash
  kubectl -n canary-upgrade get pods -l app=canary-web -o wide
  ```
  ```bash
  kubectl -n canary-upgrade describe pdb canary-web-pdb | sed -n '1,160p'
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Mantén un “latido” de tráfico para detectar interrupciones (si dejaste port-forward activo). Ejecuta el siguiente comando en la **tercera terminal**

  > **IMPORTANTE:** Verifica que en la **segunda terminal** este activo el **port-forward** sino vuelvelo a activar.
  {: .lab-note .important .compact}

  > **NOTA:** Es normal que no responda e indique que no se puede conectar, espera a que termine de actualizar el node group.
  {: .lab-note .info .compact}

  ```bash
  while true; do curl -sS http://localhost:8080 | grep -E "OK|EKS" && echo " - $(date)"; sleep 2; done
  ```
  {% include step_image.html %}

  > **IMPORTANTE:** Al finalizar corta el curl con **`CTRL+C`**
  {: .lab-note .important .compact}

- {% include step_label.html %} Confirma que el update del nodegroup finalizó y el nodegroup volvió a `ACTIVE`.

  > **NOTA:** Debera de aparecer `Successful` y `ACTIVE`.
  {: .lab-note .info .compact}

  ```bash
  aws eks describe-update --name "$CLUSTER_NAME" --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION" --update-id "$NG_UPDATE_ID" --query "update.status" --output text
  ```
  ```bash
  aws eks describe-nodegroup --cluster-name "$CLUSTER_NAME" --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION" --query "nodegroup.status" --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica salud del canario post-upgrade.

  ```bash
  kubectl -n canary-upgrade get deploy canary-web
  ```
  ```bash
  kubectl -n canary-upgrade get pods -l app=canary-web
  ```
  ```bash
  kubectl -n canary-upgrade get endpoints canary-web
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Actualización de Add-ons

Listarás add-ons instalados, actualizarás add-ons administrados por EKS (ej. vpc-cni, CoreDNS, kube-proxy) y validarás que los pods del sistema queden saludables.

#### Tarea 7.1

- {% include step_label.html %} Lista los add-ons instalados (AWS CLI y, si aplica, eksctl).

  > **NOTA:** Debes ver nombres típicos (`vpc-cni`, `coredns`, `kube-proxy`) según tu clúster. Pueden aparecer mas o menos de los mencionados.
  {: .lab-note .info .compact}

  ```bash
  aws eks list-addons --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  ```bash
  eksctl get addons --cluster "$CLUSTER_NAME" --region "$AWS_REGION" 2>/dev/null || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Genera el siguiente archivo `scripts/update-addon.yaml` para actualizar (ejemplo: `vpc-cni`) preservando conflictos.

  > **NOTA:** Ejecuta el comando de desde la carpeta raíz **lab01** y **primera terminal**
  {: .lab-note .info .compact}
  
  ```bash
  cat > scripts/update-addon.yaml <<EOF
  apiVersion: eksctl.io/v1alpha5
  kind: ClusterConfig
  metadata:
    name: ${CLUSTER_NAME}
    region: ${AWS_REGION}

  addons:
  - name: vpc-cni
    version: latest
    resolveConflicts: preserve
  EOF
  ```

- {% include step_label.html %} Ejecuta el update del add-on.

  ```bash
  eksctl update addon -f scripts/update-addon.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida la salud de pods del sistema con el siguiente comando
  
  ```bash
  kubectl -n kube-system get pods -o wide | egrep "aws-node" || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Repite para `coredns` y `kube-proxy` agregando bloques en el YAML.

  > **NOTA:** Edita el archivo **scripts/update-addon.yaml** añade bajo la sección de `addons:` y a la altura de `- name: vpc-cni`
  {: .lab-note .info .compact}

  ```yaml
  - name: coredns
    version: latest
    resolveConflicts: preserve
  - name: kube-proxy
    version: latest
    resolveConflicts: preserve
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ejecuta nuevamente el script con la actualizacion de los addons agregados.

  ```bash
  eksctl update addon -f scripts/update-addon.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida la información de los addons actualizados.

  ```bash
  kubectl -n kube-system get pods -o wide | egrep "coredns|kube-proxy|aws-node" || true
  ```
  ```bash
  eksctl get addons --cluster "$CLUSTER_NAME" --region "$AWS_REGION" 2>/dev/null || true
  ```
  {% include step_image.html %}

---

### Tarea 8. Limpieza (opcional)

Removeras toda la configuración realizada por este laboratorio.

#### Tarea 8.1

- {% include step_label.html %} Elimina el namespace del canario.

  ```bash
  kubectl delete ns canary-upgrade
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el namespace se haya eliminado correctamente.

  ```bash
  kubectl get ns | grep -n "canary-upgrade" || echo "Namespace eliminado"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el repositorio ECR del canario.

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

  > **NOTA:** El cluster tardara **7 minutos** aproximadamente en eliminarse.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el cluster se haya eliminado correctamente.

  > **NOTA:** Costos principales: 
  - EKS cobra por cluster/hora (standard vs extended support).
  - Los nodos se cobran como EC2/EBS/red.
  - ECR por almacenamiento/transferencia.
  {: .lab-note .info .compact}

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || echo "Cluster eliminado"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}