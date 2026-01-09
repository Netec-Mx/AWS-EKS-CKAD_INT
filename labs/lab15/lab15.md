---
layout: lab
title: "Práctica 15: Implementación de autenticación con IAM Roles for Service Accounts (IRSA)"
permalink: /lab15/lab15/
images_base: /labs/lab15/img
duration: "75 minutos"
objective:
  - >
    Configurar IRSA en un clúster Amazon EKS para que una aplicación en Kubernetes
    obtenga credenciales temporales y de mínimo privilegio mediante un IAM Role
    asociado a una ServiceAccount, usando OIDC + sts:AssumeRoleWithWebIdentity.
    Probarás el acceso a un bucket S3 con una política que solo permite al rol de
    IRSA (y a tu identidad administrativa), validando con pods “con” y “sin” IRSA,
    y cerrando con troubleshooting estilo CKAD (serviceaccounts, pods, logs,
    describe, env vars).
prerequisites:
  - "Cuenta AWS con permisos para: EKS, IAM y S3 (crear/leer políticas, crear roles, crear buckets)."
  - "Windows + Visual Studio Code + Terminal GitBash."
  - "Herramientas: AWS CLI v2, kubectl, eksctl, git, curl."
  - "Acceso a Internet para descargar imágenes (aws-cli) desde registries."
introduction:
  - >
    IRSA (IAM Roles for Service Accounts) permite que un Pod use un IAM Role sin
    almacenar llaves estáticas. EKS expone un issuer OIDC; el Pod recibe un token
    OIDC (JWT) proyectado en un volumen; y el SDK/CLI de AWS intercambia ese token
    por credenciales temporales con STS (AssumeRoleWithWebIdentity). Resultado:
    mínimo privilegio por ServiceAccount y rotación automática de credenciales.
slug: lab15
lab_number: 15
final_result: >
  Al finalizar, tendrás IRSA funcionando en EKS: un clúster operativo, OIDC provider
  asociado, un IAM Role con trust policy restringida al ServiceAccount (sub/aud),
  la ServiceAccount anotada con el Role ARN, y un bucket S3 protegido para permitir
  acceso únicamente al rol IRSA (y a tu identidad administrativa). Validarás en vivo
  que un Pod sin IRSA recibe AccessDenied y que un Pod con IRSA puede listar/escribir
  en S3, además de verificar internals (AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE,
  token proyectado) con troubleshooting tipo CKAD.
notes:
  - "CKAD: disciplina de validación y troubleshooting (get/describe/events/logs/exec/jsonpath)."
  - "Seguridad: evita Access Keys estáticas. IRSA entrega credenciales temporales y de mínimo privilegio."
  - "Bucket policy con condición aws:PrincipalArn para forzar que un Pod SIN IRSA falle."
references:
  - text: "Amazon EKS User Guide - IAM roles for service accounts (IRSA)"
    url: https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
  - text: "Amazon EKS User Guide - Crear un IAM OIDC provider para el clúster"
    url: https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html
  - text: "Amazon STS API Reference - AssumeRoleWithWebIdentity"
    url: https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html
  - text: "eksctl User Guide - IAM service accounts (associate-iam-oidc-provider / create iamserviceaccount)"
    url: https://docs.aws.amazon.com/eks/latest/eksctl/iamserviceaccounts.html
  - text: "Amazon S3 User Guide - Bucket policies (resource-based policies)"
    url: https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html
prev: /lab14/lab14/
next: /lab16/lab16/
---

---

## Costo (resumen)

- **EKS** cobra por clúster por hora (control plane). Revisa pricing antes de iniciar.
- **S3** cobra por almacenamiento y requests (este lab usa objetos pequeños).
- **IAM/STS** no se cobran directamente, pero forman parte del uso de AWS.

> **IMPORTANTE:** En este lab NO se usan LoadBalancers; todas las pruebas son internas.
> Aun así, **crear un clúster EKS genera costo** si no lo eliminas.
{: .lab-note .important .compact}

---

## Convenciones del laboratorio

- **Todo** se ejecuta en **GitBash** dentro de **VS Code**.
- Guarda evidencia en `outputs/` para que puedas **demostrar** el resultado.
- Si no tienes clúster, esta práctica **incluye la creación** (Tarea 2).

> **NOTA (CKAD):** Lo “evaluado” aquí es tu disciplina de troubleshooting:
> `get` → `describe` → `events` → `logs` → `exec`.
{: .lab-note .info .compact}

---

### Tarea 1. Preparación del workspace y baseline (variables + evidencias)

Crearás la carpeta del laboratorio, definirás variables reutilizables y validarás identidad AWS
herramientas y baseline.

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo del curso con un usuario con permisos administrativos.

- {% include step_label.html %} Abre **Visual Studio Code** y la terminal integrada.

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Git Bash** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Crea el directorio del laboratorio y la estructura estándar.

  ```bash
  mkdir -p lab15 && cd lab15
  mkdir -p 00-prereqs 01-eks 02-aws 03-k8s 04-tests outputs logs
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que la estructura quedó creada.

  ```bash
  find . -maxdepth 2 -type d | sort | tee outputs/00_dirs.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables base (ajusta `AWS_REGION` y `CLUSTER_NAME`).

  ```bash
  export AWS_REGION="us-west-2"
  export CLUSTER_NAME="eks-irsa-lab"
  export LAB_ID="$(date +%Y%m%d%H%M%S)"

  export NAMESPACE="irsa-lab"
  export SA_NAME="s3-writer"

  export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  export CALLER_ARN="$(aws sts get-caller-identity --query Arn --output text)"

  export ROLE_NAME="irsa-s3-writer-${CLUSTER_NAME}-${LAB_ID}"
  export POLICY_NAME="irsa-s3-writer-policy-${CLUSTER_NAME}-${LAB_ID}"

  # Bucket globalmente único (minúsculas, sin guiones bajos)
  export BUCKET_NAME="irsa-lab-${AWS_ACCOUNT_ID}-${LAB_ID}"
  ```

- {% include step_label.html %} Guarda variables para reuso (reproducible).

  ```bash
  cat > outputs/vars.env <<EOF
  AWS_REGION=$AWS_REGION
  CLUSTER_NAME=$CLUSTER_NAME
  LAB_ID=$LAB_ID
  NAMESPACE=$NAMESPACE
  SA_NAME=$SA_NAME
  AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID
  CALLER_ARN=$CALLER_ARN
  ROLE_NAME=$ROLE_NAME
  POLICY_NAME=$POLICY_NAME
  BUCKET_NAME=$BUCKET_NAME
  EOF
  ```
  ```bash
  cat outputs/vars.env | tee outputs/01_vars_echo.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica herramientas (evidencia).

  ```bash
  aws --version | tee outputs/01_aws_version.txt
  kubectl version --client=true | tee outputs/01_kubectl_version.txt
  eksctl version | tee outputs/01_eksctl_version.txt
  git --version | tee outputs/01_git_version.txt
  curl --version | head -n 2 | tee outputs/01_curl_version.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica identidad de AWS y captura evidencia (cuenta/ARN).

  ```bash
  aws sts get-caller-identity --output json | tee outputs/01_sts_identity.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Recomendado) Baseline de clusters existentes en la región.

  ```bash
  aws eks list-clusters --region "$AWS_REGION" --output json | tee outputs/01_eks_list_clusters.json
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---
### Tarea 2. Crear un clúster EKS para el laboratorio

Si ya tienes un clúster EKS funcional y `kubectl` apunta al contexto correcto, **puedes omitir esta tarea**.

> **IMPORTANTE:** Crear clúster genera costos. Elimínalo en la Tarea 8 si fue creado solo para el lab.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} Revisa tu región y AZs disponibles (evidencia).

  ```bash
  echo "AWS_REGION=$AWS_REGION" | tee outputs/02_region.txt
  ```
  ```bash
  aws ec2 describe-availability-zones --region "$AWS_REGION" \
    --query "AvailabilityZones[?State=='available'].ZoneName" --output text \
    | tee outputs/02_az_list.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el clúster con Managed Node Group (2 nodos).

  > **NOTA:** Si falla por “unsupported Kubernetes version”, vuelve a ejecutar **sin** `--version`.
  {: .lab-note .info .compact}

  ```bash
  export NODEGROUP_NAME="mng-irsa"
  export K8S_VERSION="1.33"
  ```
  ```bash
  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --version "$K8S_VERSION" \
    --managed \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-type t3.medium \
    --nodes 2 --nodes-min 2 --nodes-max 3 \
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
  {% include step_image.html %}

- {% include step_label.html %} Verifica nodos `Ready` (evidencia).

  ```bash
  kubectl get nodes -o wide | tee outputs/02_nodes_wide.txt
  ```
  ```bash
  kubectl wait --for=condition=Ready node --all --timeout=600s | tee outputs/02_wait_nodes_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Baseline de eventos recientes del clúster.

  ```bash
  kubectl get events -A --sort-by=.lastTimestamp | tail -n 30 | tee outputs/02_events_tail_baseline.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---
### Tarea 3. Habilitar y validar el IAM OIDC Provider del clúster (prerequisito IRSA)

IRSA requiere un **IAM OIDC provider** asociado al issuer OIDC del clúster EKS.
Obtendrás el issuer, verificarás si ya existe el provider y, si falta, lo crearás con `eksctl`.

> **NOTA (CKAD):** Esto entrena el hábito de **validar prerequisitos** antes de “culpar al YAML”.
{: .lab-note .info .compact}

#### Tarea 3.1

- {% include step_label.html %} Obtén el OIDC issuer del clúster y guárdalo.

  ```bash
  export OIDC_ISSUER="$(aws eks describe-cluster \
    --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.identity.oidc.issuer" --output text)"
  ```
  ```bash
  echo "OIDC_ISSUER=$OIDC_ISSUER" | tee outputs/03_oidc_issuer.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Deriva el `OIDC_ID` (último segmento del issuer).

  ```bash
  export OIDC_ID="$(echo "$OIDC_ISSUER" | awk -F'/' '{print $NF}')"
  echo "OIDC_ID=$OIDC_ID" | tee outputs/03_oidc_id.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica si ya existe un OIDC provider en IAM para este clúster.

  ```bash
  aws iam list-open-id-connect-providers \
    --query "OpenIDConnectProviderList[].Arn" --output text \
  | tr '\t' '\n' \
  | grep "$OIDC_ID" \
  | tee outputs/03_oidc_provider_arn_pre.txt

  if [ "${PIPESTATUS[2]}" -ne 0 ]; then
    echo "No existe (aún) OIDC provider (OK)" | tee outputs/03_oidc_provider_missing.txt
  fi
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asocia el OIDC provider con `eksctl`.

  ```bash
  eksctl utils associate-iam-oidc-provider \
    --cluster "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --approve | tee outputs/03_eksctl_associate_oidc.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que ahora aparece el provider ARN (evidencia).

  ```bash
  aws iam list-open-id-connect-providers \
    --query "OpenIDConnectProviderList[].Arn" --output text \
  | tr '\t' '\n' \
  | grep "$OIDC_ID" \
  | tee outputs/03_oidc_provider_arn.txt

  if [ "${PIPESTATUS[2]}" -ne 0 ]; then
    echo "No existe (aún) OIDC provider (OK)" | tee outputs/03_oidc_provider_missing.txt
  fi
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

### Tarea 4. Crear S3 + IAM Policy/Role + ServiceAccount con IRSA

Crearás un bucket S3 para pruebas, una **IAM policy** de mínimo privilegio restringida a ese bucket, y una **ServiceAccount** con un **IAM Role** asociado (IRSA). Después aplicarás una **bucket policy** que permita acceso **solo** a:

- 1) tu identidad administrativa actual (`CALLER_ARN`) y
- 2) el `ROLE_ARN` de IRSA.

Así, un Pod **sin IRSA** debe fallar con **AccessDenied**.

#### Tarea 4.1 — Namespace + bucket S3

- {% include step_label.html %} Crea el namespace del laboratorio (idempotente) y captura evidencia.

  ```bash
  kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
  ```
  {% include step_image.html %}
  ```bash
  kubectl get ns "$NAMESPACE" -o wide | tee outputs/04_ns_created.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el bucket S3 (maneja el caso `us-west-2`) y guarda evidencia.

  ```bash
  if [ "$AWS_REGION" = "us-east-1" ]; then
    aws s3api create-bucket --bucket "$BUCKET_NAME" --region "$AWS_REGION"
  else
    aws s3api create-bucket --bucket "$BUCKET_NAME" --region "$AWS_REGION" \
      --create-bucket-configuration LocationConstraint="$AWS_REGION"
  fi
  ```
  {% include step_image.html %}
  ```bash
  aws s3api get-bucket-location --bucket "$BUCKET_NAME" --output json | tee outputs/04_bucket_location.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Recomendado) Bloquea acceso público del bucket (higiene de seguridad).

  ```bash
  aws s3api put-public-access-block \
    --bucket "$BUCKET_NAME" \
    --public-access-block-configuration '{
      "BlockPublicAcls": true,
      "IgnorePublicAcls": true,
      "BlockPublicPolicy": true,
      "RestrictPublicBuckets": true
    }'
  ```
  {% include step_image.html %}
  ```bash
  aws s3api get-public-access-block --bucket "$BUCKET_NAME" --output json | tee outputs/04_public_access_block.json
  ```
  {% include step_image.html %}

#### Tarea 4.2 — IAM Policy (mínimo privilegio)

- {% include step_label.html %} Crea el documento de policy (solo tu bucket) y guarda evidencia.

  ```bash
  cat > 02-aws/iam-policy-s3.json <<EOF
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "ListBucket",
        "Effect": "Allow",
        "Action": ["s3:ListBucket"],
        "Resource": ["arn:aws:s3:::$BUCKET_NAME"]
      },
      {
        "Sid": "RWObjects",
        "Effect": "Allow",
        "Action": ["s3:GetObject","s3:PutObject","s3:DeleteObject"],
        "Resource": ["arn:aws:s3:::$BUCKET_NAME/*"]
      }
    ]
  }
  EOF
  ```
  ```bash
  cat 02-aws/iam-policy-s3.json | tee outputs/04_iam_policy_s3.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea la policy en IAM y guarda el ARN.

  > **NOTA:** Si falla por nombre duplicado, cambia `POLICY_NAME` (agrega sufijo) y reintenta.
  {: .lab-note .important .compact}

  ```bash
  export POLICY_ARN="$(aws iam create-policy \
    --policy-name "$POLICY_NAME" \
    --policy-document file://02-aws/iam-policy-s3.json \
    --query 'Policy.Arn' --output text)"
  ```
  ```bash
  echo "$POLICY_ARN" | tee outputs/04_policy_arn.txt
  ```
  {% include step_image.html %}

#### Tarea 4.3 — ServiceAccount + Role con IRSA (eksctl)

- {% include step_label.html %} Crea la ServiceAccount con rol IRSA y adjunta la policy.

  ```bash
  eksctl create iamserviceaccount \
    --cluster "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --namespace "$NAMESPACE" \
    --name "$SA_NAME" \
    --role-name "$ROLE_NAME" \
    --attach-policy-arn "$POLICY_ARN" \
    --override-existing-serviceaccounts \
    --approve | tee outputs/04_eksctl_create_iamserviceaccount.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que la ServiceAccount tenga la anotación `eks.amazonaws.com/role-arn`.

  ```bash
  kubectl -n "$NAMESPACE" get sa "$SA_NAME" -o yaml | sed -n '1,140p' | tee outputs/04_sa_yaml.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Extrae el `ROLE_ARN` desde la anotación y guárdalo.

  ```bash
  export ROLE_ARN="$(kubectl -n "$NAMESPACE" get sa "$SA_NAME" \
    -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}')"
  echo "$ROLE_ARN" | tee outputs/04_role_arn.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Recomendado) Inspecciona trust policy del rol (sub/aud) para confirmar que apunta a tu namespace/SA.

  > **Qué buscar:** condiciones que restrinjan el `sub` a `system:serviceaccount:<namespace>:<sa>` y `aud` a `sts.amazonaws.com`.
  {: .lab-note .info .compact}
  
  ```bash
  aws iam get-role --role-name "$ROLE_NAME" \
    --query 'Role.AssumeRolePolicyDocument' --output json \
    | tee outputs/04_trust_policy.json
  ```
  {% include step_image.html %}

#### Tarea 4.4 — Bucket policy (permitir solo tu identidad y el rol IRSA)

- {% include step_label.html %} Crea bucket policy con condición `aws:PrincipalArn`.

  > **Por qué así:** algunos Pods podrían obtener credenciales del **node role** (IMDS). Esta policy fuerza que **solo** tu identidad (`CALLER_ARN`) y el rol IRSA (`ROLE_ARN`) tengan acceso.
  {: .lab-note .info .compact}

  ```bash
  cat > 02-aws/s3-bucket-policy.json <<EOF
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowOnlyAdminAndIRSA",
        "Effect": "Allow",
        "Principal": { "AWS": "arn:aws:iam::$AWS_ACCOUNT_ID:root" },
        "Action": ["s3:ListBucket","s3:GetObject","s3:PutObject","s3:DeleteObject"],
        "Resource": [
          "arn:aws:s3:::$BUCKET_NAME",
          "arn:aws:s3:::$BUCKET_NAME/*"
        ],
        "Condition": {
          "ArnLike": {
            "aws:PrincipalArn": [
              "$CALLER_ARN",
              "$ROLE_ARN",
              "arn:aws:sts::$AWS_ACCOUNT_ID:assumed-role/$ROLE_NAME/*"
            ]
          }
        }
      }
    ]
  }
  EOF
  ```
  ```bash
  cat 02-aws/s3-bucket-policy.json | tee outputs/04_bucket_policy.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Aplica la bucket policy y valida que quedó puesta.

  ```bash
  aws s3api put-bucket-policy --bucket "$BUCKET_NAME" --policy file://02-aws/s3-bucket-policy.json
  ```
  ```bash
  aws s3api get-bucket-policy --bucket "$BUCKET_NAME" --query 'Policy' --output text \
    | head -c 260; echo \
    | tee outputs/04_bucket_policy_head.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---

### Tarea 5. Probar acceso desde Pods: SIN IRSA vs CON IRSA

Crearás dos pods con AWS CLI:

- `awscli-noirsa` usa ServiceAccount `default` (sin anotación IRSA) → **debe fallar** con `AccessDenied` en S3.
- `awscli-irsa` usa la ServiceAccount con IRSA → **debe funcionar** (STS + S3).

> **NOTA (CKAD):** Comparación perfecta para troubleshooting: *mismo namespace, misma imagen, mismos comandos*; solo cambia la ServiceAccount.
{: .lab-note .info .compact}

#### Tarea 5.1 — Crear Pods (sleep) para exec y evidencia

- {% include step_label.html %} Crea Pod **SIN IRSA** (ServiceAccount `default`) y espera `Ready`.

  ```bash
  cat > 03-k8s/01_pod_noirsa.yaml <<EOF
  apiVersion: v1
  kind: Pod
  metadata:
    name: awscli-noirsa
    namespace: $NAMESPACE
    labels:
      app: awscli-test
      mode: noirsa
  spec:
    serviceAccountName: default
    containers:
    - name: awscli
      image: public.ecr.aws/aws-cli/aws-cli:latest
      command: ["sh","-c","sleep 3600"]
      env:
      - name: AWS_REGION
        value: "$AWS_REGION"
  EOF
  ```
  ```bash
  kubectl apply -f 03-k8s/01_pod_noirsa.yaml
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" wait --for=condition=Ready pod/awscli-noirsa --timeout=180s
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" get pod awscli-noirsa -o wide | tee outputs/05_pod_noirsa_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea Pod **CON IRSA** (ServiceAccount `$SA_NAME`) y espera `Ready`.

  ```bash
  cat > 03-k8s/02_pod_irsa.yaml <<EOF
  apiVersion: v1
  kind: Pod
  metadata:
    name: awscli-irsa
    namespace: $NAMESPACE
    labels:
      app: awscli-test
      mode: irsa
  spec:
    serviceAccountName: $SA_NAME
    containers:
    - name: awscli
      image: public.ecr.aws/aws-cli/aws-cli:latest
      command: ["sh","-c","sleep 3600"]
      env:
      - name: AWS_REGION
        value: "$AWS_REGION"
      - name: AWS_STS_REGIONAL_ENDPOINTS
        value: "regional"
  EOF
  ```
  ```bash
  kubectl apply -f 03-k8s/02_pod_irsa.yaml
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" wait --for=condition=Ready pod/awscli-irsa --timeout=180s
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" get pod awscli-irsa -o wide | tee outputs/05_pod_irsa_ready.txt
  ```
  {% include step_image.html %}

#### Tarea 5.2 — Ejecutar pruebas comparables

- {% include step_label.html %} Prueba en pod **SIN IRSA**: STS + S3 (S3 debe fallar con `AccessDenied`).

  ```bash
  kubectl -n "$NAMESPACE" exec -it awscli-noirsa -- sh -lc "
    echo '== STS identity (puede ser node role) ==';
    aws sts get-caller-identity || true;

    echo '== S3 LIST (debe FALLAR: AccessDenied) ==';
    aws s3 ls s3://$BUCKET_NAME 2>&1 || true;

    echo '== S3 PUT (debe FALLAR: AccessDenied) ==';
    echo 'no-irsa' > /tmp/noirsa.txt;
    aws s3 cp /tmp/noirsa.txt s3://$BUCKET_NAME/test/noirsa.txt 2>&1 || true
  " | tee outputs/05_noirsa_test.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba en pod **CON IRSA**: STS + S3 (debe funcionar).

  ```bash
  kubectl -n "$NAMESPACE" exec -it awscli-irsa -- sh -lc "
    echo '== STS identity (debe mostrar assumed-role) ==';
    aws sts get-caller-identity;

    echo '== S3 PUT (debe OK) ==';
    echo 'hola-irsa' > /tmp/hello.txt;
    aws s3 cp /tmp/hello.txt s3://$BUCKET_NAME/test/hello.txt;

    echo '== S3 LIST (debe OK) ==';
    aws s3 ls s3://$BUCKET_NAME/test/
  " | tee outputs/05_irsa_test.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Validación externa (desde tu máquina) para confirmar que el objeto existe.

  ```bash
  aws s3 ls "s3://$BUCKET_NAME/test/" | tee outputs/05_external_s3_ls.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

---

### Tarea 6. Verificación interna de IRSA + troubleshooting

Confirmarás IRSA “por dentro” del pod (env vars + token proyectado) y ejecutarás troubleshooting típico
(`describe/events/logs`).

#### Tarea 6.1 — Verificar internals dentro del pod

- {% include step_label.html %} Verifica variables IRSA y la presencia del token (sin imprimirlo completo).

  ```bash
  kubectl -n "$NAMESPACE" exec -it awscli-irsa -- sh -lc '
    echo "== ENV (IRSA) ==";
    env | grep -E "AWS_ROLE_ARN|AWS_WEB_IDENTITY_TOKEN_FILE|AWS_REGION|AWS_STS_REGIONAL_ENDPOINTS" || true;

    echo "== TOKEN DIR ==";
    ls -l /var/run/secrets/eks.amazonaws.com/serviceaccount/ || true;

    echo "== TOKEN FILE ==";
    echo "AWS_WEB_IDENTITY_TOKEN_FILE=$AWS_WEB_IDENTITY_TOKEN_FILE";
    test -f "$AWS_WEB_IDENTITY_TOKEN_FILE" && echo "OK: token presente" || echo "ERROR: token NO encontrado";
    echo "Token bytes:"; wc -c "$AWS_WEB_IDENTITY_TOKEN_FILE" || true
  ' | tee outputs/06_irsa_internals.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma la identidad STS dentro del Pod con IRSA (debe ser assumed-role).

  ```bash
  kubectl -n "$NAMESPACE" exec -it awscli-irsa -- sh -lc '
    echo "== STS identity ==";
    aws sts get-caller-identity
  ' | tee outputs/06_sts_identity_in_pod.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma el wiring: Pod → ServiceAccount → anotación role-arn.

  ```bash
  kubectl -n "$NAMESPACE" get pod awscli-irsa -o jsonpath='{.spec.serviceAccountName}{"\n"}' \
    | tee outputs/06_pod_sa_name.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" get sa "$SA_NAME" -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}{"\n"}' \
    | tee outputs/06_sa_role_arn.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Recomendado) Revisa eventos del namespace.

  ```bash
  kubectl -n "$NAMESPACE" get events --sort-by=.lastTimestamp | tail -n 60 \
    | tee outputs/06_events_tail.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Recomendado) Revisa el Pod a detalle (volúmenes proyectados, SA, errores).

  ```bash
  kubectl -n "$NAMESPACE" describe pod awscli-irsa | sed -n '1,260p' \
    | tee outputs/06_describe_pod_irsa.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Checklist de fallos comunes (guía rápida):

  - 1) **OIDC provider faltante** → `outputs/03_oidc_provider_arn.txt` está vacío.
  - 2) **SA sin anotación** → `outputs/04_sa_yaml.txt` no muestra `eks.amazonaws.com/role-arn`.
  - 3) **Pod no usa la SA correcta** → `outputs/06_pod_sa_name.txt` no es `$SA_NAME`.
  - 4) **Trust policy incorrecta** → revisa `outputs/04_trust_policy.json` (sub/aud/namespace/SA).
  - 5) **La app ignora IRSA** por credenciales previas en la chain → revisa `env` en `outputs/06_irsa_internals.txt`.
  - 6) **Sin salida a STS/S3** (NAT/VPC endpoints/DNS) → verás timeouts en `outputs/05_irsa_test.txt`.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r6 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r6 %}

---

### Tarea 7. Checklist final (evidencia estilo examen)

Ejecutarás un checklist para dejar evidencia del estado: namespace, ServiceAccount, Pods,
identidad STS dentro del Pod con IRSA y objetos en S3.

#### Tarea 7.1

- {% include step_label.html %} Estado de recursos Kubernetes del laboratorio.

  ```bash
  kubectl -n "$NAMESPACE" get sa,pods -o wide | tee outputs/07_k8s_resources.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" get pod awscli-irsa -o jsonpath='{.spec.serviceAccountName}{"\n"}' \
    | tee outputs/07_sa_used_by_pod.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Evidencia de identidad STS dentro del Pod con IRSA (debe ser assumed-role).

  ```bash
  kubectl -n "$NAMESPACE" exec -it awscli-irsa -- aws sts get-caller-identity \
    | tee outputs/07_sts_identity_in_pod.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Evidencia de objetos de prueba en S3.

  ```bash
  aws s3 ls "s3://$BUCKET_NAME/test/" | tee outputs/07_s3_test_prefix_ls.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r7 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r7 %}

---

### Tarea 8. Limpieza del laboratorio (recomendada)

Eliminarás recursos para evitar costos y dejar el entorno limpio.
Si creaste el clúster en la Tarea 2, también lo eliminarás.

> **IMPORTANTE:** Borra primero objetos del bucket antes de borrar el bucket.
{: .lab-note .important .compact}

#### Tarea 8.1

- {% include step_label.html %} (Si reiniciaste terminal) Recarga variables guardadas.

  ```bash
  set -a
  source outputs/vars.env
  set +a
  export POLICY_ARN="$(cat outputs/04_policy_arn.txt)"
  ```

- {% include step_label.html %} Borra pods del laboratorio.

  ```bash
  kubectl -n "$NAMESPACE" delete pod awscli-noirsa awscli-irsa --ignore-not-found \
    | tee outputs/08_delete_pods.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Borra la iamserviceaccount con eksctl (revierte SA + rol asociado).

  ```bash
  eksctl delete iamserviceaccount \
    --cluster "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --namespace "$NAMESPACE" \
    --name "$SA_NAME" \
    | tee outputs/08_eksctl_delete_iamserviceaccount.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Borra la policy IAM creada (usa el ARN guardado).

  ```bash
  aws iam delete-policy --policy-arn "$POLICY_ARN" | tee outputs/08_delete_policy.txt
  ```

- {% include step_label.html %} Vacía y borra el bucket.

  ```bash
  aws s3 rm "s3://$BUCKET_NAME" --recursive | tee outputs/08_s3_rm_recursive.txt
  ```
  ```bash
  aws s3api delete-bucket --bucket "$BUCKET_NAME" --region "$AWS_REGION" \
    | tee outputs/08_delete_bucket.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Borra el namespace del laboratorio.

  ```bash
  kubectl delete ns "$NAMESPACE" | tee outputs/08_delete_namespace.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get ns | grep -w "$NAMESPACE" && echo "Namespace aún existe (espera terminación)" \
    || echo "OK: namespace eliminado" | tee outputs/08_verify_ns_deleted.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Elimina el clúster si fue creado solo para esta práctica.

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