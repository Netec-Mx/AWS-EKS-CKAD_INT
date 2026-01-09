---
layout: lab
title: "Práctica 7: Configuración de escalado de nodos con Karpenter"
permalink: /lab7/lab7/
images_base: /labs/lab7/img
duration: "50 minutos"
objective: >
  Configurar Karpenter en un clúster Amazon EKS para aprovisionar (scale-up) y retirar (scale-down) nodos automáticamente en función de Pods Pending/Unschedulable, validando el comportamiento con NodePool/EC2NodeClass, requisitos de scheduling y consolidación, usando prácticas alineadas a CKAD (requests/limits, nodeSelector/affinity, taints/tolerations y troubleshooting con eventos).
prerequisites:
  - "Cuenta AWS con permisos para: EKS, IAM, EC2, CloudFormation y SQS"
  - "Windows + Visual Studio Code + Terminal GitBash"
  - "Herramientas: AWS CLI v2, kubectl, eksctl, helm 3, jq, git"
  - "Docker Desktop (Docker local)"
  - "Conectividad a internet para bajar charts/manifiestos"
introduction:
  - "Karpenter observa el estado del scheduler (Pods Pending/Unschedulable) y crea nodos con las características necesarias (instancia, zona, capacity-type, etc.). En lugar de administrar múltiples NodeGroups, defines la intención con NodePools y la configuración AWS con EC2NodeClass; Karpenter se encarga del aprovisionamiento y la consolidación de capacidad."
slug: lab7
lab_number: 7
final_result: >
  Al finalizar, Karpenter quedará instalado y configurado con NodePool/EC2NodeClass, capaz de crear nodos cuando haya Pods Unschedulable y de consolidar/retirar nodos cuando la carga desaparece; todo validado con evidencia (pods Pending→Running, nuevos nodes/NodeClaims, logs del controller y Events) y con prácticas de scheduling/depuración alineadas a CKAD.
notes:
  - "CKAD: enfócate en por qué un Pod queda Pending: requests/limits, taints/tolerations, nodeSelector/affinity y Events (FailedScheduling)."
  - "Ojo con costos: Karpenter puede crear instancias rápidamente. Usa NodePool.spec.limits como cinturón de seguridad."
  - "Mejor en dev/stage. NO ejecutes esta práctica en producción."
references:
  - text: "Karpenter Provider AWS - Releases (verifica la versión que usarás)"
    url: https://github.com/aws/karpenter-provider-aws/releases
  - text: "Karpenter - Getting Started with Karpenter (AWS Provider)"
    url: https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/
  - text: "Karpenter - CloudFormation (IAM + Interruption Queue / SQS) recomendado"
    url: https://karpenter.sh/docs/reference/cloudformation/
  - text: "Karpenter - NodePools (conceptos y ejemplos)"
    url: https://karpenter.sh/docs/concepts/nodepools/
  - text: "Karpenter - NodeClasses / EC2NodeClass"
    url: https://karpenter.sh/docs/concepts/nodeclasses/
  - text: "EKS Workshop - Karpenter (conceptos, NodePool/NodeClass y consolidación)"
    url: https://www.eksworkshop.com/docs/fundamentals/compute/karpenter
  - text: "AWS - Amazon EKS pricing (costo del control plane)"
    url: https://aws.amazon.com/eks/pricing/
  - text: "AWS - EKS Cluster Access Manager / Access Entries (referencia API)"
    url: https://docs.aws.amazon.com/eks/latest/APIReference/API_CreateAccessEntry.html
prev: /lab6/lab6/
next: /lab8/lab8/
---

## Costo (resumen)

- **EKS** cobra por clúster por hora (estándar vs **Extended Support**). Revisa la página de pricing antes de iniciar el laboratorio.
- **Karpenter** no se cobra como servicio separado, pero **sí** generará recursos como **EC2 instances**, **EBS**, y (por arquitectura recomendada) **SQS** para interrupciones.

> **IMPORTANTE:** Durante la prueba de scale-up, Karpenter puede crear instancias rápidamente. Mantén el demo corto y elimina recursos al final.
{: .lab-note .important .compact}

---

### Tarea 1. Preparación del workspace y variables

Crearás la carpeta de la práctica, validarás herramientas, autenticarás AWS y dejarás listas variables de entorno reutilizables para operar EKS/Karpenter desde GitBash.

> **NOTA (CKAD):** Esta práctica entrena scheduling y troubleshooting: `requests/limits`, `taints/tolerations`, `nodeSelector/affinity`, `kubectl describe`, Events (`FailedScheduling`) y lectura de logs.
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo del curso con un usuario con permisos administrativos.

- {% include step_label.html %} Abre **Visual Studio Code**.

- {% include step_label.html %} Abre la terminal integrada en VS Code (ícono de terminal en la parte superior derecha).

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Git Bash** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de estar en la carpeta raíz de tu repositorio/laboratorios (por ejemplo **`labs-eks-ckad`**).

  > Si vienes de otra práctica, usa `cd ..` hasta volver a la raíz.
  {: .lab-note .info .compact}

- {% include step_label.html %} Crea la carpeta del laboratorio y la estructura estándar.

  ```bash
  mkdir lab07 && cd lab07
  mkdir -p manifests iam logs outputs scripts notes
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que la estructura de carpetas quedó creada (esto evita errores de rutas durante la práctica).

  ```bash
  find . -maxdepth 2 -type d | sort
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea un archivo `.env` con variables base (recomendado para reproducibilidad).

  ```bash
  cat > .env <<'EOF'
  export AWS_REGION="us-west-2"

  # Namespace donde correrá Karpenter (aislado del resto)
  export KARPENTER_NAMESPACE="karpenter"

  # Versión del provider AWS de Karpenter (ver releases)
  export KARPENTER_VERSION="1.7.4"
  EOF
  ```
  ```bash
  source .env
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que tus CLIs responden correctamente (suele fallar por PATH o instalaciones incompletas).

  ```bash
  aws --version
  ```
  ```bash
  kubectl version --client=true
  ```
  ```bash
  eksctl version
  ```
  ```bash
  jq --version
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica tu identidad en AWS (debe mostrar `Account` y `Arn`; si falla, corrige credenciales/perfil antes de continuar).

  ```bash
  aws sts get-caller-identity
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda tu `AWS_ACCOUNT_ID` para reutilizarlo más adelante.

  ```bash
  export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  ```
  ```bash
  echo "AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. (Opcional) Crear un clúster EKS para esta práctica

Crea un clúster EKS con Managed Node Group usando `eksctl`. **Omite esta tarea** si ya tienes un clúster y conoces su `CLUSTER_NAME`.

> **IMPORTANTE:** Crear clúster genera costos. Úsalo solo si no cuentas con un clúster de laboratorio.
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

- {% include step_label.html %} Define variables del clúster de laboratorio.

  ```bash
  export CLUSTER_NAME="eks-karpen-lab"
  export NODEGROUP_NAME="mng-1"
  export K8S_VERSION="1.33"
  ```

- {% include step_label.html %} Crea el clúster con un Managed Node Group.

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

- {% include step_label.html %} Verifica que el clúster quedó `ACTIVE` desde AWS (esto confirma que la creación terminó).

  ```bash
  aws eks describe-cluster \
    --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.status" --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que existe al menos un NodeGroup asociado al clúster (esto indica que habrá nodos para el sistema).

  ```bash
  aws eks list-nodegroups \
    --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --output table
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegura que el OIDC provider esté asociado (requisito para IRSA del controller de Karpenter).

  ```bash
  eksctl utils associate-iam-oidc-provider \
    --cluster "$CLUSTER_NAME" --region "$AWS_REGION" --approve
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el issuer OIDC exista.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
  --query "cluster.identity.oidc.issuer" --output text
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Conectar kubectl al clúster y validar conectividad

Actualizarás tu kubeconfig y verificarás que puedes listar nodos. Esta tarea es obligatoria tanto si usas un clúster existente como si lo creaste en la Tarea 2.

#### Tarea 3.1

- {% include step_label.html %} Actualiza tu kubeconfig hacia el clúster objetivo (esto “apunta” kubectl al lugar correcto).

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma el contexto actual (evita aplicar recursos en el clúster equivocado).

  ```bash
  kubectl config current-context
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el control plane responde (si falla, no continúes: primero arregla acceso/red/VPN/credenciales).

  ```bash
  kubectl cluster-info
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista nodos y confirma que están `Ready` (sin nodos, Karpenter no podrá operar correctamente).

  ```bash
  kubectl get nodes -o wide
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Prerrequisitos AWS/EKS para Karpenter

Prepararás el clúster para que los nodos creados por Karpenter puedan unirse, y para que Karpenter descubra subnets/SG y procese eventos de interrupción vía SQS (arquitectura recomendada).

#### Tarea 4.1 — Verificar modo de autenticación (Access Entries)

- {% include step_label.html %} Verifica el modo de autenticación del clúster (esto define si usarás Access Entries o aws-auth).

  ```bash
  aws eks describe-cluster \
    --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query 'cluster.accessConfig.authenticationMode' --output text
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si el modo es `CONFIG_MAP`, actualízalo a `API_AND_CONFIG_MAP` para poder usar Access Entries (recomendado).

  > **IMPORTANTE:** Es normal que aparezca el mensaje de error, porque ya existe el metodo de autenticación correcto. avanza al siguiente paso.
  {: .lab-note .important .compact}

  ```bash
  aws eks update-cluster-config \
    --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --access-config authenticationMode=API_AND_CONFIG_MAP
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que el clúster regresó a estado `ACTIVE` tras el cambio (si está `UPDATING`, espera).

  ```bash
  aws eks describe-cluster \
    --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query 'cluster.status' --output text
  ```
  {% include step_image.html %}

#### Tarea 4.2 — Tags de descubrimiento en subnets y security groups

- {% include step_label.html %} Obtén los Subnet IDs del clúster y aplícales el tag de discovery `karpenter.sh/discovery=$CLUSTER_NAME` (esto habilita selección por tags en EC2NodeClass).

  > **Tip:** En entornos serios, taggea solo subnets privadas. Para el laboratorio, taggear los subnets del clúster es suficiente.
  {: .lab-note .info .compact}

  ```bash
  SUBNETS="$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.resourcesVpcConfig.subnetIds[]" --output text)"
  ```
  ```bash
  aws ec2 create-tags --region "$AWS_REGION" \
    --resources $SUBNETS \
    --tags Key=karpenter.sh/discovery,Value="$CLUSTER_NAME"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén el **Cluster Security Group** y aplícale el mismo discovery tag (EC2NodeClass puede seleccionar SGs por tags).

  ```bash
  CLUSTER_SG="$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)"
  ```
  ```bash
  aws ec2 create-tags --region "$AWS_REGION" \
    --resources "$CLUSTER_SG" \
    --tags Key=karpenter.sh/discovery,Value="$CLUSTER_NAME"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Comprueba que los tags quedaron aplicados (si no aparecen, Karpenter no podrá descubrir recursos).

  ```bash
  aws ec2 describe-tags --region "$AWS_REGION" \
    --filters "Name=key,Values=karpenter.sh/discovery" "Name=value,Values=$CLUSTER_NAME" \
    --query "Tags[].{ResourceId:ResourceId,Key:Key,Value:Value}" --output table
  ```
  {% include step_image.html %}

#### Tarea 4.3 — Crear IAM/SQS con CloudFormation (bootstrap oficial)

- {% include step_label.html %} Descarga el template oficial de CloudFormation para tu versión de Karpenter (esto crea NodeRole, policies y la SQS para interrupciones).

  ```bash
  export CFN_TEMPLATE="iam/karpenter-cloudformation.yaml"
  ```
  ```bash
  curl -fsSL \
    "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml" \
    > "$CFN_TEMPLATE"
  ```
  ```bash
  wc -l "$CFN_TEMPLATE"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Despliega el stack (crea/actualiza recursos IAM + SQS). Este paso es idempotente.

  > **NOTA:** La creación del stack puede durar **2 minutos**
  {: .lab-note .info .compact}

  ```bash
  aws cloudformation deploy \
    --stack-name "Karpenter-${CLUSTER_NAME}" \
    --template-file "$CFN_TEMPLATE" \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameter-overrides "ClusterName=${CLUSTER_NAME}"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el stack quedó `CREATE_COMPLETE` o `UPDATE_COMPLETE` (si falla, revisa permisos o conflictos de nombres).

  ```bash
  aws cloudformation describe-stacks \
    --stack-name "Karpenter-${CLUSTER_NAME}" \
    --query "Stacks[0].StackStatus" --output text
  ```
  {% include step_image.html %}

#### Tarea 4.4 — Crear Access Entry para el Node Role (nodos creados por Karpenter)

- {% include step_label.html %} Define el ARN del Node Role creado por CloudFormation (por convención: `KarpenterNodeRole-$CLUSTER_NAME`).

  ```bash
  export NODE_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}"
  echo "NODE_ROLE_ARN=$NODE_ROLE_ARN"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el Access Entry para el rol de nodo (si ya existe, el comando puede fallar: es aceptable en el lab).

  ```bash
  aws eks create-access-entry \
    --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --principal-arn "$NODE_ROLE_ARN" \
    --type EC2_LINUX || echo "Access entry ya existe o no es necesario (verifica con list-access-entries)."
  ```
  {% include step_image.html %}

- {% include step_label.html %} Comprueba que el Access Entry aparece en el clúster (sin esto, los nodos nuevos pueden no unirse).

  ```bash
  aws eks list-access-entries \
    --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --output table
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Instalar Karpenter con Helm (IRSA)

Instalarás los CRDs y el controlador de Karpenter usando el chart OCI, habilitando IRSA a través de anotación del ServiceAccount con el rol IAM del controller.

#### Tarea 5.1 — Obtener ARN/IDs (rol del controller + cola SQS)

- {% include step_label.html %} Exporta los outputs del stack para identificar el rol IAM del controller y el nombre de la SQS.

  ```bash
  export STACK="Karpenter-${CLUSTER_NAME}"
  echo $STACK
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén la SQS Queue URL del stack y deriva el QueueName.

  ```bash
  KARPENTER_SQS_QUEUE_URL="$(aws cloudformation list-stack-resources --stack-name "$STACK" \
    --query "StackResourceSummaries[?ResourceType=='AWS::SQS::Queue'].PhysicalResourceId | [0]" \
    --output text)"
  ```
  ```bash
  export KARPENTER_SQS_QUEUE="$(echo "$KARPENTER_SQS_QUEUE_URL" | awk -F/ '{print $NF}')"
  ```
  ```bash
  echo "KARPENTER_SQS_QUEUE_URL=$KARPENTER_SQS_QUEUE_URL"
  echo "KARPENTER_SQS_QUEUE=$KARPENTER_SQS_QUEUE"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén el ARN de la ManagedPolicy creada por el stack (controller policy).

  ```bash
  export KARPENTER_CONTROLLER_POLICY_ARN="$(aws cloudformation list-stack-resources --stack-name "$STACK" \
    --query "StackResourceSummaries[?ResourceType=='AWS::IAM::ManagedPolicy'].PhysicalResourceId | [0]" \
    --output text)"
  ```
  ```bash
  echo "KARPENTER_CONTROLLER_POLICY_ARN=$KARPENTER_CONTROLLER_POLICY_ARN"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén el endpoint del clúster (valor recomendado para Helm).

  ```bash
  export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
  --query "cluster.endpoint" --output text)"
  ```
  ```bash
  echo "CLUSTER_ENDPOINT=$CLUSTER_ENDPOINT"
  ```
  {% include step_image.html %}

#### Tarea 5.2 — Crear ServiceAccount con IRSA (rol del controller)

- {% include step_label.html %} Crea/actualiza el ServiceAccount karpenter con IRSA (eksctl crea el role y deja la anotación).

  > **NOTA:** La creación puede tardar **2 minutos**
  {: .lab-note .info .compact}

  ```bash
  eksctl create iamserviceaccount \
  --cluster "$CLUSTER_NAME" --region "$AWS_REGION" \
  --namespace "$KARPENTER_NAMESPACE" \
  --name karpenter \
  --attach-policy-arn "$KARPENTER_CONTROLLER_POLICY_ARN" \
  --approve \
  --override-existing-serviceaccounts
  ```
  {% include step_image.html %}

- {% include step_label.html %} Extrae el Role ARN desde la anotación del ServiceAccount.

  ```bash
  export KARPENTER_IAM_ROLE_ARN="$(kubectl -n "$KARPENTER_NAMESPACE" get sa karpenter \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}')"
  ```
  ```bash
  echo "KARPENTER_IAM_ROLE_ARN=$KARPENTER_IAM_ROLE_ARN"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el SA quedó anotado (prueba de IRSA).

  ```bash
  kubectl -n "$KARPENTER_NAMESPACE" get sa karpenter -o yaml | sed -n '1,120p'
  ```
  {% include step_image.html %}

#### Tarea 5.3 — Instalar CRDs y controlador

- {% include step_label.html %} Asegura que Helm no tenga sesión autenticada contra `public.ecr.aws` (pull público).

  ```bash
  helm registry logout public.ecr.aws >/dev/null 2>&1 || true
  ```

- {% include step_label.html %} Instala/actualiza el chart de **CRDs** (esto evita errores por CRDs faltantes).

  ```bash
  helm upgrade --install karpenter-crd oci://public.ecr.aws/karpenter/karpenter-crd \
    --version "${KARPENTER_VERSION}" \
    --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
    --wait
  ```
  {% include step_image.html %}

- {% include step_label.html %} Instala/actualiza el controlador Karpenter con IRSA (anota el ServiceAccount con el rol IAM).

  ```bash
  helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
    --version "${KARPENTER_VERSION}" \
    --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
    --set "settings.clusterName=${CLUSTER_NAME}" \
    --set "settings.clusterEndpoint=${CLUSTER_ENDPOINT}" \
    --set "settings.interruptionQueue=${KARPENTER_SQS_QUEUE}" \
    --set serviceAccount.create=false \
    --set serviceAccount.name=karpenter \
    --set controller.resources.requests.cpu=1 \
    --set controller.resources.requests.memory=1Gi \
    --set controller.resources.limits.cpu=1 \
    --set controller.resources.limits.memory=1Gi \
    --wait
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el deployment esté Ready **(si no está Ready, aún no continúes con NodePool/NodeClass).**

  ```bash
  kubectl get deploy -n "${KARPENTER_NAMESPACE}"
  ```
  ```bash
  kubectl rollout status deploy/karpenter -n "${KARPENTER_NAMESPACE}"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el ServiceAccount tiene la anotación del rol IAM (esto prueba que IRSA quedó aplicado).

  ```bash
  kubectl get sa -n "${KARPENTER_NAMESPACE}" karpenter -o yaml | sed -n '1,120p'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa logs recientes buscando errores típicos (AccessDenied, STS, errores de SQS).

  ```bash
  kubectl logs -n "${KARPENTER_NAMESPACE}" deploy/karpenter --tail=200 | tee logs/karpenter_tail_install.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Crear EC2NodeClass y NodePool

Definirás la **política de aprovisionamiento** (NodePool) y la **configuración AWS** (EC2NodeClass) para que Karpenter sepa qué nodos puede crear y dónde (subnets/SG/AMI/rol).

> **NOTA (CKAD - muy relevante):** Aquí practicas scheduling real: `requirements`, `taints/tolerations`, `requests/limits` y cómo provocan Pods **Pending**.
{: .lab-note .info .compact}

#### Tarea 6.1

- {% include step_label.html %} Crea el manifiesto `manifests/karpenter-default.yaml` (NodePool + EC2NodeClass).

  ```bash
  cat > manifests/karpenter-default.yaml <<EOF
  apiVersion: karpenter.sh/v1
  kind: NodePool
  metadata:
    name: default
  spec:
    template:
      metadata:
        labels:
          workload: karpenter
      spec:
        # CKAD: taints/tolerations controlan qué Pods pueden correr aquí
        taints:
          - key: workload
            value: karpenter
            effect: NoSchedule

        requirements:
          - key: kubernetes.io/arch
            operator: In
            values: ["amd64"]
          - key: kubernetes.io/os
            operator: In
            values: ["linux"]
          - key: karpenter.sh/capacity-type
            operator: In
            values: ["on-demand"]
          - key: karpenter.k8s.aws/instance-category
            operator: In
            values: ["c","m","r"]
          - key: karpenter.k8s.aws/instance-generation
            operator: Gt
            values: ["2"]

        nodeClassRef:
          group: karpenter.k8s.aws
          kind: EC2NodeClass
          name: default

        expireAfter: 720h

    # Cinturón de seguridad (control de costos)
    limits:
      cpu: 100

    disruption:
      consolidationPolicy: WhenEmptyOrUnderutilized
      consolidateAfter: 1m
  ---
  apiVersion: karpenter.k8s.aws/v1
  kind: EC2NodeClass
  metadata:
    name: default
  spec:
    role: "KarpenterNodeRole-${CLUSTER_NAME}"
    amiSelectorTerms:
      - alias: "al2023@latest"
    subnetSelectorTerms:
      - tags:
          karpenter.sh/discovery: "${CLUSTER_NAME}"
    securityGroupSelectorTerms:
      - tags:
          karpenter.sh/discovery: "${CLUSTER_NAME}"
  EOF
  ```

- {% include step_label.html %} Aplica el manifiesto (crea NodePool/EC2NodeClass).

  ```bash
  kubectl apply -f manifests/karpenter-default.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que ambos recursos existen (si no existen, revisa CRDs o namespace).

  ```bash
  kubectl get nodepool
  ```
  ```bash
  kubectl get ec2nodeclass
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inspecciona con `describe` para detectar errores de selectors/tags (aquí se ve si NO encontró subnets/SG por tags).

  ```bash
  kubectl describe ec2nodeclass default | sed -n '1,220p' | tee outputs/ec2nodeclass_describe.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl describe nodepool default | sed -n '1,220p' | tee outputs/nodepool_describe.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa eventos del namespace de Karpenter para señales tempranas de problemas (por ejemplo, IAM o discovery).

  ```bash
  kubectl get events -n "${KARPENTER_NAMESPACE}" --sort-by=.lastTimestamp | tail -n 40
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Probar scale-up y scale-down (con troubleshooting estilo CKAD)

Desplegarás una carga con `requests` altos que no quepa en los nodos actuales para forzar Pods Pending/Unschedulable y disparar aprovisionamiento. Luego reducirás a 0 réplicas y observarás consolidación/terminación.

#### Tarea 7.1 — Desplegar carga que fuerza aprovisionamiento

- {% include step_label.html %} Crea el namespace para el ejemplo. Ahora crea el Deployment de prueba (incluye toleration para el taint del NodePool).

  ```bash
  kubectl create ns karpenter-demo || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora crea el Deployment de prueba (incluye toleration para el taint del NodePool).

  ```bash
  cat > manifests/scale-demo.yaml <<'EOF'
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: scale-demo
    namespace: karpenter-demo
  spec:
    replicas: 10
    selector:
      matchLabels:
        app: scale-demo
    template:
      metadata:
        labels:
          app: scale-demo
      spec:
        # CKAD: toleration requerida por el taint del NodePool
        tolerations:
          - key: "workload"
            operator: "Equal"
            value: "karpenter"
            effect: "NoSchedule"

        containers:
          - name: pause
            image: public.ecr.aws/eks-distro/kubernetes/pause:3.9
            resources:
              requests:
                cpu: "900m"
                memory: "512Mi"
  EOF
  ```

- {% include step_label.html %} Aplica el manifiesto.

  ```bash
  kubectl apply -f manifests/scale-demo.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} **Observa Pods:** al inicio es normal ver `Pending` mientras Karpenter decide y crea capacidad.

  ```bash
  kubectl get pods -n karpenter-demo -w
  ```
  {% include step_image.html %}

- {% include step_label.html %} **En otra terminal,** observa la aparición de nuevos Nodes (evidencia directa de scale-up).

  ```bash
  kubectl get nodes -o wide -w
  ```
  {% include step_image.html %}

#### Tarea 7.2 — Diagnóstico (si hay Pending) + evidencia

- {% include step_label.html %} Si los Pods siguen Pending, describe un Pod para ver la razón (busca `FailedScheduling`, `Insufficient cpu`, taints, etc.).

  ```bash
  kubectl get pods -n karpenter-demo
  ```
  {% include step_image.html %}
  ```bash
  kubectl describe pod -n karpenter-demo -l app=scale-demo | sed -n '1,260p' | tee outputs/pods_describe_scale_demo.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa eventos del namespace (aquí se ve el motivo exacto del scheduler y la evolución del caso).

  ```bash
  kubectl get events -n karpenter-demo --sort-by=.lastTimestamp | tail -n 60 | tee outputs/events_karpenter_demo.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista NodeClaims para ver recursos creados por Karpenter (puede variar por versión/config).

  ```bash
  kubectl get nodeclaims 2>/dev/null || echo "No hay CRD NodeClaim visible (depende de versión/config)."
  ```
  {% include step_image.html %}

#### Tarea 7.3 — Scale-down + consolidación

- {% include step_label.html %} Baja réplicas a cero (esto elimina demanda y habilita consolidación).

  ```bash
  kubectl scale deploy scale-demo -n karpenter-demo --replicas=0
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa que los Pods terminan y que, tras `consolidateAfter: 1m`, Karpenter comienza a retirar nodos (evidencia de scale-down).

  ```bash
  kubectl get pods -n karpenter-demo -w
  ```

- {% include step_label.html %} **En otra terminal,** observa cómo cambian los nodos (puede tardar algunos minutos según la política y drenado).

  ```bash
  kubectl get nodes -o wide -w
  ```

- {% include step_label.html %} Captura estado final (deberías ver 0 réplicas y menos nodos que durante el pico).

  ```bash
  kubectl get deploy -n karpenter-demo
  ```
  ```bash
  kubectl get nodes -o wide
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

## Tarea 8. Limpieza y control de costos

Eliminarás la carga de prueba y, si este clúster es solo de laboratorio, podrás desinstalar Karpenter y eliminar el stack CloudFormation.

### Tarea 8.1

- {% include step_label.html %} Elimina el workload de prueba y el namespace.

  ```bash
  kubectl delete -f manifests/scale-demo.yaml --ignore-not-found
  ```
  ```bash
  kubectl delete ns karpenter-demo --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que ya no quedan Pods del demo.

  ```bash
  kubectl get pods -n karpenter-demo 2>/dev/null || echo "OK: namespace/pods del demo eliminados"
  ```

- {% include step_label.html %} (Opcional) Elimina NodePool/EC2NodeClass si NO los reutilizarás.

  ```bash
  kubectl delete -f manifests/karpenter-default.yaml --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Desinstala Karpenter (controller + CRDs chart).

  ```bash
  helm uninstall karpenter -n "${KARPENTER_NAMESPACE}" || true
  ```
  ```bash
  helm uninstall karpenter-crd -n "${KARPENTER_NAMESPACE}" || true
  ```
  ```bash
  kubectl delete ns "${KARPENTER_NAMESPACE}" --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que no quedaron namespaces de práctica.

  ```bash
  kubectl get ns | egrep "karpenter-demo|${KARPENTER_NAMESPACE}" || echo "OK: namespaces removidos"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el clúster creado por `eksctl`.

  > **NOTA:** El cluster tardara **9 minutos** aproximadamente en eliminarse.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Elimina el stack CloudFormation (IAM/SQS).

  ```bash
  aws cloudformation delete-stack --stack-name "Karpenter-${CLUSTER_NAME}"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que ya no queden stacks o esten en **DELETE_IN_PROGRESS**

  ```bash
  aws cloudformation describe-stacks \
  --query "Stacks[].{StackName:StackName,Status:StackStatus,Created:CreationTime,Updated:LastUpdatedTime}" \
  --output table
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