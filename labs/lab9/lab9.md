---
layout: lab
title: "Práctica 9: Aprovisionamiento de recursos en AWS con Crossplane desde EKS"
permalink: /lab9/lab9/
images_base: /labs/lab9/img
duration: "60 minutos"
objective:
  - "Instalar Crossplane en un clúster Amazon EKS, autenticar hacia AWS usando IRSA y aprovisionar (y eliminar) un S3 Bucket y una SQS Queue declarativamente con manifiestos Kubernetes listos para GitOps, validando condiciones SYNCED/READY y evidencias con AWS CLI; con troubleshooting estilo CKAD (CRDs, namespaces, ServiceAccounts, describe/events/logs)."
prerequisites:
  - "Amazon EKS accesible con kubectl."
  - "Windows + Visual Studio Code + Terminal GitBash."
  - "Herramientas: AWS CLI v2, kubectl, helm v3.2+, eksctl."
  - "(Opcional) jq y git."
  - "Permisos AWS: IAM/EKS (OIDC + IRSA), S3 y SQS (crear/eliminar), y lectura de CloudTrail/STS para ver identidad."
introduction:
  - "Crossplane convierte Kubernetes en un control plane capaz de crear y reconciliar infraestructura externa (AWS) usando CRDs. En lugar de click-ops, declaras el estado deseado en YAML; Crossplane + Providers lo aplican y lo mantienen. En este laboratorio instalarás Crossplane con Helm, instalarás Providers de AWS (S3/SQS), configurarás autenticación IRSA (sin llaves estáticas) y crearás recursos reales en AWS desde manifiestos Kubernetes."
slug: lab9
lab_number: 9
final_result: |
  Al finalizar habrás instalado Crossplane en EKS, habilitado autenticación a AWS vía IRSA para los Providers, y creado/eliminado un Bucket S3 y una Queue SQS declarativamente con CRDs; verificando condiciones (SYNCED/READY), eventos/logs y existencia real en AWS con AWS CLI. Habrás practicado habilidades CKAD (CRDs, control loops, namespaces, ServiceAccounts e investigación de fallos con kubectl describe/events/logs).
notes:
  - "CKAD: práctica de control loops y troubleshooting (CRDs + reconcile + describe/events/logs)."
  - "Best practice: IRSA evita credenciales estáticas. En producción aplica least privilege (políticas mínimas) y control de cambios (GitOps)."
  - "Costo: Crossplane no se cobra como servicio; corre sobre tus nodos EKS. S3/SQS cobran por uso. Elimina recursos al final para evitar cargos."
references:
  - text: "Crossplane Docs - Install Crossplane (Helm)"
    url: https://docs.crossplane.io/latest/get-started/install/
  - text: "Crossplane Docs - Providers (concepto)"
    url: https://docs.crossplane.io/v2.1/packages/providers/
  - text: "Upbound Docs - Provider Authentication (AWS: IRSA/WebIdentity)"
    url: https://docs.upbound.io/providers/provider-aws/authentication/
  - text: "AWS Docs - eksctl: IAM Roles for Service Accounts (IRSA)"
    url: https://docs.aws.amazon.com/eks/latest/eksctl/iamserviceaccounts.html
  - text: "Upbound Marketplace - provider-aws-s3"
    url: https://marketplace.upbound.io/providers/upbound/provider-aws-s3
  - text: "Upbound Marketplace - provider-aws-sqs"
    url: https://marketplace.upbound.io/providers/upbound/provider-aws-sqs
prev: /lab8/lab8/
next: /lab10/lab10/
---

---

### Tarea 1. Preparar la carpeta de la práctica y verificar el contexto

Crearás la carpeta del laboratorio, abrirás **GitBash** dentro de **VS Code**, definirás variables reutilizables (región/cluster) y validarás que `aws` y `kubectl` apuntan al lugar correcto antes de instalar Crossplane.

> **NOTA (CKAD):** La rutina base de troubleshooting es: confirmar contexto → describir recursos → leer eventos → revisar logs del controlador.
{: .lab-note .info .compact}

#### Tarea 1.1

- {% include step_label.html %} Inicia sesión en tu equipo de trabajo con un usuario con permisos para operar AWS CLI y kubectl.

- {% include step_label.html %} Abre **Visual Studio Code**.

- {% include step_label.html %} Abre la terminal integrada en VS Code.

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Git Bash** como terminal.

  {% include step_image.html %}

- {% include step_label.html %} Ubícate en la raíz de tu repo/labs (por ejemplo `labs-eks-ckad`).

  > **Nota:** Si vienes de otro lab, usa `cd ..` hasta llegar a la raíz.
  {: .lab-note .info .compact}

- {% include step_label.html %} Crea el directorio del lab y su estructura.

  ```bash
  mkdir lab09 && cd lab09
  mkdir -p manifests/01-crossplane manifests/02-providers manifests/03-irsa manifests/04-managed scripts outputs logs iam
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma la estructura de carpetas creada.

  ```bash
  find . -maxdepth 3 -type d | sort
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables base (ajusta a tu entorno).

  ```bash
  export AWS_REGION="us-west-2"
  export CLUSTER_NAME="eks-crossplane-lab"
  ```

- {% include step_label.html %} Guarda variables en `scripts/vars.env` para poder recargarlas rápido (sin credenciales).

  ```bash
  cat > scripts/vars.env <<EOF
  export AWS_REGION="${AWS_REGION}"
  export CLUSTER_NAME="${CLUSTER_NAME}"
  EOF
  ```

- {% include step_label.html %} Carga variables desde el archivo (si reabriste terminal).

  ```bash
  source scripts/vars.env
  ```

- {% include step_label.html %} Verifica identidad AWS (evita operar en la cuenta equivocada).

  ```bash
  aws sts get-caller-identity --output table
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma la región efectiva (debe ser la misma donde está tu clúster EKS).

  ```bash
  aws configure get region || true
  ```
  ```bash
  echo "AWS_REGION=$AWS_REGION"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear un clúster EKS para la práctica (opcional)

Crearás un clúster EKS con Managed Node Group usando `eksctl`. **Omite esta tarea** si ya tienes un clúster y `kubectl` se conecta correctamente.

> **IMPORTANTE:** Crear un clúster genera costos (EKS + EC2). Elimínalo al final si es solo laboratorio.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} (Recomendado) Lista versiones disponibles en tu región para elegir `K8S_VERSION`.

  ```bash
  eksctl utils describe-cluster-versions strings --region "$AWS_REGION" \
    | jq -r '.clusterVersions[].ClusterVersion'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegúrate de que la versión elegida aparece en la lista, usaremos la version **`1.33`**.

- {% include step_label.html %} Define variables del clúster.

  ```bash
  export CLUSTER_NAME="eks-crossplane-lab"
  export NODEGROUP_NAME="mng-1"
  export K8S_VERSION="1.33"
  ```

- {% include step_label.html %} Valida que las variables quedaron correctas.

  ```bash
  echo "AWS_REGION=$AWS_REGION"
  echo "CLUSTER_NAME=$CLUSTER_NAME"
  echo "NODEGROUP_NAME=$NODEGROUP_NAME"
  echo "K8S_VERSION=$K8S_VERSION"
  ```
  {% include step_image.html %}

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

- {% include step_label.html %} Verifica que el control plane responde (si falla, no continúes: primero arregla acceso/red/VPN/credenciales).

  ```bash
  kubectl cluster-info
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que existe al menos un NodeGroup asociado al clúster (esto indica que habrá nodos para el sistema).

  ```bash
  aws eks list-nodegroups \
    --cluster-name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --output table
  ```
  {% include step_image.html %}

- {% include step_label.html %} Actualiza tu kubeconfig hacia el clúster objetivo (esto “apunta” kubectl al lugar correcto).

  ```bash
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista nodos y confirma que están `Ready` (sin nodos, Karpenter no podrá operar correctamente).

  ```bash
  kubectl get nodes -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica desde AWS que el clúster está `ACTIVE` (evidencia).

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.status" --output text | tee outputs/02_cluster_status.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Instalar Crossplane en EKS (Helm)

Instalarás Crossplane en `crossplane-system` usando Helm y validarás que los pods quedan **Running/Ready**. También verificarás que exista el CRD **DeploymentRuntimeConfig** (lo usarás para inyectar IRSA sin depender de nombres exactos de ServiceAccounts).

> **NOTA (CKAD):** Instalas un controlador/aparecen CRDs/el controlador reconcilia recursos. Ese “loop” es el corazón del examen.
{: .lab-note .info .compact}

#### Tarea 3.1

- {% include step_label.html %} Agrega el repo Helm oficial y actualiza índices.

  ```bash
  helm repo add crossplane-stable https://charts.crossplane.io/stable
  helm repo update
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora instala Crossplane.

  ```bash
  helm install crossplane --namespace crossplane-system --create-namespace crossplane-stable/crossplane
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa pods del namespace (deben crearse varios componentes).

  ```bash
  kubectl -n crossplane-system get pods -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera a que todos los pods estén listos.

  ```bash
  kubectl -n crossplane-system wait --for=condition=Ready pod --all --timeout=240s
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda la evidencia de los nodos.

  > **NOTA:** Recuerda revisar siempre el directorio **outputs** donde se guardan las evidencias.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n crossplane-system get pods -o wide | tee outputs/03_crossplane_pods.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma deployments instalados (evidencia útil cuando algo no reconcilia).

  ```bash
  kubectl -n crossplane-system get deploy -o wide | tee outputs/03_crossplane_deploys.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que existe el CRD `DeploymentRuntimeConfig` (lo usaremos para IRSA).

  ```bash
  kubectl get crd deploymentruntimeconfigs.pkg.crossplane.io >/dev/null 2>&1 && echo "OK: DeploymentRuntimeConfig CRD presente" || echo "WARN: No veo el CRD deploymentruntimeconfigs.pkg.crossplane.io"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Instalar Providers de AWS (S3 y SQS)

Instalarás los Providers que agregan CRDs para administrar recursos AWS desde Kubernetes. Usaremos Providers oficiales de Upbound para **S3** y **SQS** y validaremos `INSTALLED=True` y `HEALTHY=True`.

> **Nota:** Si tu organización fija versiones, pinéalas. En laboratorio, define una versión conocida y evita “latest”.
{: .lab-note .info .compact}

#### Tarea 4.1

- {% include step_label.html %} Define una versión de Providers (ejemplo) y guárdala para reproducibilidad.

  ```bash
  export PROVIDER_VER="v2.3.0"
  echo "PROVIDER_VER=$PROVIDER_VER" | tee outputs/04_provider_version.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea manifiesto del Provider para S3.

  ```bash
  cat > manifests/02-providers/provider-aws-s3.yaml <<EOF
  apiVersion: pkg.crossplane.io/v1
  kind: Provider
  metadata:
    name: provider-aws-s3
  spec:
    package: xpkg.upbound.io/upbound/provider-aws-s3:${PROVIDER_VER}
  EOF
  ```

- {% include step_label.html %} Crea manifiesto del Provider para SQS.

  ```bash
  cat > manifests/02-providers/provider-aws-sqs.yaml <<EOF
  apiVersion: pkg.crossplane.io/v1
  kind: Provider
  metadata:
    name: provider-aws-sqs
  spec:
    package: xpkg.upbound.io/upbound/provider-aws-sqs:${PROVIDER_VER}
  EOF
  ```

- {% include step_label.html %} Aplica ambos Providers.

  ```bash
  kubectl apply -f manifests/02-providers/provider-aws-s3.yaml
  ```
  ```bash
  kubectl apply -f manifests/02-providers/provider-aws-sqs.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa el avance de instalación (espera a `HEALTHY=True`).

  ```bash
  kubectl get providers -o wide
  ```
  {% include step_image.html %}

- {% include step_label.html %} Repite los comandos hasta ver `INSTALLED=True` y `HEALTHY=True` (evidencia).

  ```bash
  kubectl get providers -o wide | tee outputs/04_providers_status.txt
  ```
  ```bash
  kubectl get providerrevisions -o wide | tee outputs/04_providerrevisions.txt
  ```

- {% include step_label.html %} Descubre recursos nuevos (CRDs) que quedaron disponibles.

  ```bash
  kubectl api-resources | egrep -i "ProviderConfig|Bucket|Queue" || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si alguno queda `HEALTHY=False`, captura logs del Crossplane core (diagnóstico).

  > **NOTA:** Puedes ignorar los **Warnings** no deben de afectar la ejecución de la practica.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n crossplane-system logs deploy/crossplane --tail=200 | tee logs/04_crossplane_tail.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Configurar autenticación a AWS con IRSA

Configurarás IRSA para que los pods de los Providers obtengan credenciales AWS mediante un IAM Role asociado a su ServiceAccount (sin llaves estáticas). Como los Providers pueden crear ServiceAccounts con sufijos variables, usaremos un **trust policy** con `StringLike` para “provider-aws-*” en `crossplane-system`, y un **DeploymentRuntimeConfig** para inyectar la anotación `eks.amazonaws.com/role-arn`.

> **Plan A (recomendado):** `DeploymentRuntimeConfig` + `runtimeConfigRef` en Provider.  
> **Plan B (alterno):** si NO tienes `DeploymentRuntimeConfig`, usa un Secret con WebIdentity/IRSA no aplica; en ese caso tendrás que usar credenciales estáticas (no recomendado) o actualizar Crossplane/Providers.  
{: .lab-note .important .compact}

#### Tarea 5.1 — Asegurar OIDC (requisito de IRSA)

- {% include step_label.html %} Asocia el OIDC provider (idempotente) para permitir WebIdentity en ServiceAccounts.

  ```bash
  eksctl utils associate-iam-oidc-provider \
    --cluster "$CLUSTER_NAME" --region "$AWS_REGION" --approve
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica el issuer OIDC del clúster (evidencia).

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --query "cluster.identity.oidc.issuer" --output text | tee outputs/05_oidc_issuer.txt
  ```
  {% include step_image.html %}

#### Tarea 5.2 — Crear IAM Role para Providers (trust policy)

- {% include step_label.html %} Obtén `AWS_ACCOUNT_ID` y construye el ARN del OIDC provider.

  ```bash
  export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
  export OIDC_ISSUER="$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query "cluster.identity.oidc.issuer" --output text)"
  export OIDC_PROVIDER="${OIDC_ISSUER#https://}"
  export OIDC_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
  ```
  ```bash
  echo "AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID" | tee outputs/05_account.txt
  echo "OIDC_PROVIDER=$OIDC_PROVIDER" | tee outputs/05_oidc_provider.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el trust policy (ServiceAccounts del namespace `crossplane-system` con nombre `provider-aws-*`).

  ```bash
  cat > iam/trust-policy.json <<EOF
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": { "Federated": "${OIDC_ARN}" },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringLike": {
            "${OIDC_PROVIDER}:sub": "system:serviceaccount:crossplane-system:provider-aws-*"
          },
          "StringEquals": {
            "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
          }
        }
      }
    ]
  }
  EOF
  ```

- {% include step_label.html %} Crea el rol IAM (si ya existe, reutilízalo) y captura el ARN.

  ```bash
  export IRSA_ROLE_NAME="xp-crossplane-aws-lab9"
  ```
  ```bash
  aws iam get-role --role-name "$IRSA_ROLE_NAME" >/dev/null 2>&1 || aws iam create-role --role-name "$IRSA_ROLE_NAME" --assume-role-policy-document file://iam/trust-policy.json --output table
  ```
  {% include step_image.html %}
  ```bash
  export IRSA_ROLE_ARN="$(aws iam get-role --role-name "$IRSA_ROLE_NAME" --query Role.Arn --output text)"
  echo "IRSA_ROLE_ARN=$IRSA_ROLE_ARN" | tee outputs/05_irsa_role_arn.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Adjunta permisos (efectivo para laboratorio). En producción, reemplaza por políticas mínimas.

  ```bash
  aws iam attach-role-policy --role-name "$IRSA_ROLE_NAME" --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
  aws iam attach-role-policy --role-name "$IRSA_ROLE_NAME" --policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess
  ```
  {% include step_image.html %}

#### Tarea 5.3 — Inyectar IRSA en los Providers con DeploymentRuntimeConfig

- {% include step_label.html %} Confirma nuevamente que existe el CRD `DeploymentRuntimeConfig` (si falla, documenta y pasa al Plan B).

  ```bash
  kubectl get crd deploymentruntimeconfigs.pkg.crossplane.io >/dev/null 2>&1 && echo "OK: puedo usar DeploymentRuntimeConfig" || echo "WARN: No existe DeploymentRuntimeConfig (Plan B requerido)"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea un `DeploymentRuntimeConfig` que anota el ServiceAccount de los Providers con tu role ARN.

  ```bash
  cat > manifests/03-irsa/runtimeconfig-irsa.tpl.yaml <<'EOF'
  apiVersion: pkg.crossplane.io/v1beta1
  kind: DeploymentRuntimeConfig
  metadata:
    name: aws-irsa
  spec:
    serviceAccountTemplate:
      metadata:
        annotations:
          eks.amazonaws.com/role-arn: ${IRSA_ROLE_ARN}
  EOF
  ```

- {% include step_label.html %} Aplica el runtime config usando `envsubst` y guarda evidencia.

  ```bash
  envsubst < manifests/03-irsa/runtimeconfig-irsa.tpl.yaml | kubectl apply -f -
  ```
  ```bash
  kubectl get deploymentruntimeconfig aws-irsa -o yaml | sed -n '1,120p' | tee outputs/05_runtimeconfig.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asocia el runtime config a cada Provider (patch).

  ```bash
  kubectl patch provider provider-aws-s3 --type='merge' -p '{"spec":{"runtimeConfigRef":{"name":"aws-irsa"}}}'
  ```
  ```bash
  kubectl patch provider provider-aws-sqs --type='merge' -p '{"spec":{"runtimeConfigRef":{"name":"aws-irsa"}}}'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Identifica deployments de cada Provider (los nombres pueden variar por versión, por eso lo buscamos).

  ```bash
  kubectl -n crossplane-system get deploy -o name | grep -E "provider-aws-s3|provider-aws-sqs|upbound-provider-family-aws" | tee outputs/05_provider_deploy_names.txt
  ```
  {% include step_image.html %}
  ```bash
  export S3_DEPLOY="$(kubectl -n crossplane-system get deploy -o name | grep provider-aws-s3 | head -n 1)"
  export SQS_DEPLOY="$(kubectl -n crossplane-system get deploy -o name | grep provider-aws-sqs | head -n 1)"
  export AWS_FAMILY_DEPLOY="$(kubectl -n crossplane-system get deploy -o name | grep upbound-provider-family-aws | head -n 1 || true)"
  ```
  ```bash
  echo "S3_DEPLOY=$S3_DEPLOY" | tee -a outputs/05_provider_deploys_vars.txt
  echo "SQS_DEPLOY=$SQS_DEPLOY" | tee -a outputs/05_provider_deploys_vars.txt
  echo "AWS_FAMILY_DEPLOY=$AWS_FAMILY_DEPLOY" | tee -a outputs/05_provider_deploys_vars.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera a que los pods del Provider estén Ready después del patch (puede disparar nueva revision).

  ```bash
  kubectl -n crossplane-system rollout status "$S3_DEPLOY" --timeout=240s
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n crossplane-system rollout status "$SQS_DEPLOY" --timeout=240s
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n crossplane-system get pods -o wide | egrep "provider-aws-s3|provider-aws-sqs|upbound-provider-family-aws" | tee outputs/05_provider_pods_ready.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el ServiceAccount usado por el Provider tiene la anotación `eks.amazonaws.com/role-arn`.

  ```bash
  export S3_SA="$(kubectl -n crossplane-system get "$S3_DEPLOY" -o jsonpath='{.spec.template.spec.serviceAccountName}')"
  export SQS_SA="$(kubectl -n crossplane-system get "$SQS_DEPLOY" -o jsonpath='{.spec.template.spec.serviceAccountName}')"
  ```
  ```bash
  echo "S3_SA=$S3_SA" | tee outputs/05_provider_sas_vars.txt
  echo "SQS_SA=$SQS_SA" | tee -a outputs/05_provider_sas_vars.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n crossplane-system get sa "$S3_SA" -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}{""}' | tee outputs/05_sa_annotation_s3.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n crossplane-system get sa "$SQS_SA" -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}{""}' | tee outputs/05_sa_annotation_sqs.txt
  ```
  {% include step_image.html %}

#### Tarea 5.3.1 — Hotfix (“token file name cannot be empty”)

> **IMPORTANTE:** Garantiza que exista el token de IRSA montado en el pod del provider (automount) y que el role ARN esté anotado en el ServiceAccount real que está usando el provider.  
{: .lab-note .important .compact}

- {% include step_label.html %} Fuerza la anotación de role ARN + automount token en *todas* las ServiceAccounts de Providers (idempotente).

  ```bash
  for SA in $(kubectl -n crossplane-system get sa -o name | egrep -i 'provider-aws|upbound|upjet' | cut -d/ -f2); do
    echo "=== Fix SA: $SA ==="
    kubectl -n crossplane-system annotate sa "$SA" eks.amazonaws.com/role-arn="$IRSA_ROLE_ARN" --overwrite
    kubectl -n crossplane-system patch sa "$SA" -p '{"automountServiceAccountToken": true}' >/dev/null 2>&1 || true
    kubectl -n crossplane-system get sa "$SA"       -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}{"  automount="}{.automountServiceAccountToken}{""}'
  done | tee outputs/05_hotfix_sa_irsa.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Reinicia deployments de providers para que tomen el cambio.

  ```bash
  kubectl -n crossplane-system get deploy -o name | egrep -i 'provider-aws|upbound|upjet' | while read -r D; do
    echo "=== Restart $D ==="
    kubectl -n crossplane-system rollout restart "$D"
    kubectl -n crossplane-system rollout status "$D"
  done | tee outputs/05_hotfix_restart_providers.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida en el POD del provider que existen variables y montaje de WebIdentity (sin “exec”, solo `describe`).

  ```bash
  POD="$(kubectl -n crossplane-system get pods -o name | egrep -i 'provider-aws-s3|provider-aws-sqs|upbound-provider-family-aws' | head -n1 | cut -d/ -f2)"
  echo "POD=$POD" | tee outputs/05_hotfix_pod_selected.txt
  kubectl -n crossplane-system describe pod "$POD" | egrep -n "Service Account:|AWS_WEB_IDENTITY_TOKEN_FILE|AWS_ROLE_ARN|/var/run/secrets/eks.amazonaws.com/serviceaccount" -i -A2 -B2 | tee outputs/05_hotfix_pod_irsa_snippet.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que el webhook del provider (9443) responde (sin esto, puedes ver errores de conversion webhook).

  ```bash
  kubectl -n crossplane-system get svc | egrep -i 'provider-aws-s3|provider-aws-sqs|upbound-provider-family-aws' | tee outputs/05_provider_svcs_9443.txt
  ```
  ```bash
  kubectl -n crossplane-system get endpoints | egrep -i 'provider-aws-s3|provider-aws-sqs|upbound-provider-family-aws' | tee outputs/05_provider_endpoints_9443.txt || true
  ```
  {% include step_image.html %}

#### Tarea 5.4 — Crear ProviderConfig usando IRSA

- {% include step_label.html %} Crea el `ProviderConfig` default (los recursos usarán este config).

  ```bash
  cat > manifests/03-irsa/providerconfig-default.yaml <<'EOF'
  apiVersion: aws.upbound.io/v1beta1
  kind: ProviderConfig
  metadata:
    name: default
  spec:
    credentials:
      source: IRSA
  EOF
  ```

- {% include step_label.html %} Aplica y confirma que existe (evidencia).

  ```bash
  kubectl apply -f manifests/03-irsa/providerconfig-default.yaml
  ```
  ```bash
  kubectl get providerconfig.aws.upbound.io default -o yaml | sed -n '1,140p' | tee outputs/05_providerconfig_default.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Nota de verificación.

  > **NOTA:** Es normal ver `status: {}` en el ProviderConfig. Se vuelve relevante cuando un Managed Resource lo usa. Si luego ves `CannotConnectToProvider` o `token file name cannot be empty`, regresa a la **Tarea 5.3.1 (Hotfix)**.
  {: .lab-note .info .compact}

- {% include step_label.html %} Si aparece algún error de autenticación, revisa logs del Provider (S3/SQS) buscando `AccessDenied` o `AssumeRoleWithWebIdentity`.

  > **NOTA:** Si no detectas errores continua con las tareas.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n crossplane-system logs "$S3_DEPLOY" --tail=200 | tee logs/05_provider_s3_tail.txt || true
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n crossplane-system logs "$SQS_DEPLOY" --tail=200 | tee logs/05_provider_sqs_tail.txt || true
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Aprovisionar S3 Bucket y SQS Queue declarativamente, validar y limpiar

Crearás un Bucket S3 y una Queue SQS como CRs de Kubernetes. Crossplane los creará en AWS y podrás validar:

- En Kubernetes: condiciones `SYNCED=True` y `READY=True`, eventos y describe.
- En AWS: existencia real con AWS CLI.

> **NOTA (CKAD):** Si un recurso no se crea, casi siempre la evidencia está en `kubectl describe` + `Events`.
{: .lab-note .info .compact}

#### Tarea 6.1

- {% include step_label.html %} Genera un sufijo único (evita colisiones, especialmente en S3).

  ```bash
  RAND="$(openssl rand -hex 3)"
  echo "RAND=$RAND" | tee outputs/06_rand.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define nombres únicos y guárdalos para reusar en validaciones/cleanup.

  ```bash
  export BUCKET_NAME="xp-lab9-${AWS_ACCOUNT_ID}-${RAND}"
  export QUEUE_NAME="xp-lab9-queue-${AWS_ACCOUNT_ID}-${RAND}"
  ```
  ```bash
  echo "BUCKET_NAME=$BUCKET_NAME" | tee outputs/06_names.txt
  echo "QUEUE_NAME=$QUEUE_NAME" | tee -a outputs/06_names.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el manifiesto del Bucket S3.

  ```bash
  cat > manifests/04-managed/s3-bucket.yaml <<EOF
  apiVersion: s3.aws.upbound.io/v1beta1
  kind: Bucket
  metadata:
    name: ${BUCKET_NAME}
  spec:
    forProvider:
      region: ${AWS_REGION}
    providerConfigRef:
      name: default
  EOF
  ```

- {% include step_label.html %} Crea el manifiesto de la cola SQS.

  ```bash
  cat > manifests/04-managed/sqs-queue.yaml <<EOF
  apiVersion: sqs.aws.upbound.io/v1beta1
  kind: Queue
  metadata:
    name: ${QUEUE_NAME}
  spec:
    forProvider:
      name: ${QUEUE_NAME}
      region: ${AWS_REGION}
    providerConfigRef:
      name: default
  EOF
  ```

- {% include step_label.html %} Aplica ambos recursos (Crossplane iniciará reconciliación).

  ```bash
  kubectl apply -f manifests/04-managed/s3-bucket.yaml
  kubectl apply -f manifests/04-managed/sqs-queue.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa el estado inicial en Kubernetes **(puede tardar 1–3 minutos)**.

  ```bash
  kubectl get buckets.s3.aws.upbound.io
  ```
  ```bash
  kubectl get queues.sqs.aws.upbound.io
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inspecciona cada recurso con `describe` y guarda evidencia (condiciones + eventos).

  ```bash
  kubectl describe buckets.s3.aws.upbound.io "$BUCKET_NAME" | sed -n '1,240p' | tee outputs/06_describe_bucket.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl describe queues.sqs.aws.upbound.io "$QUEUE_NAME" | sed -n '1,240p' | tee outputs/06_describe_queue.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Extrae condiciones con JSONPath para confirmar `SYNCED/READY` filtrado.

  ```bash
  kubectl get buckets.s3.aws.upbound.io "$BUCKET_NAME" -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"  "}{end}{""}' | tee outputs/06_bucket_conditions.txt
  ```
  ```bash
  kubectl get queues.sqs.aws.upbound.io "$QUEUE_NAME" -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"  "}{end}{""}' | tee outputs/06_queue_conditions.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura los eventos recientes del clúster (pista de permisos, nombres inválidos, región, etc.).

  ```bash
  kubectl get events --sort-by=.lastTimestamp | tail -n 40 | tee outputs/06_events_tail.txt
  ```

- {% include step_label.html %} Valida existencia real en AWS con CLI (fuente de verdad externa).

  ```bash
  aws s3api head-bucket --bucket "$BUCKET_NAME" && echo "S3 OK" | tee outputs/06_aws_s3_ok.txt
  ```
  {% include step_image.html %}
  ```bash
  aws sqs get-queue-url --queue-name "$QUEUE_NAME" --region "$AWS_REGION" | tee outputs/06_aws_sqs_url.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si algún recurso queda `READY=False`, revisa logs del Provider específico (S3 o SQS) y busca `AccessDenied`/`Invalid`/`Throttling`.

  > **NOTA:** Si ves mensajes tipo `CannotConnectToProvider` o `token file name cannot be empty`, regresa a la **Tarea 5.3.1 (Hotfix)** y luego vuelve a consultar `get/describe`.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n crossplane-system logs "$S3_DEPLOY" --tail=250 | tee logs/06_provider_s3_tail.txt || true
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n crossplane-system logs "$SQS_DEPLOY" --tail=250 | tee logs/06_provider_sqs_tail.txt || true
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

--- 

### Tarea 7 — Limpieza (recomendada)

Si este clúster es solo para laboratorio y quieres dejarlo “limpio”, puedes desinstalar providers, Crossplane y (si aplica) el clúster.

> **IMPORTANTE:** Haz esto solo si nadie más usa Crossplane en este clúster.
{: .lab-note .important .compact}

#### Tarea 7.1

- {% include step_label.html %} Elimina los CRs (Crossplane eliminará recursos externos en AWS durante la reconciliación).

  ```bash
  kubectl delete -f manifests/04-managed/sqs-queue.yaml
  ```
  ```bash
  kubectl delete -f manifests/04-managed/s3-bucket.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que ya no aparecen en Kubernetes.

  ```bash
  kubectl get buckets.s3.aws.upbound.io | tee outputs/06_after_delete_buckets.txt
  ```
  ```bash
  kubectl get queues.sqs.aws.upbound.io | tee outputs/06_after_delete_queues.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica en AWS que ya no existan (puede tardar por reconciliación).

  ```bash
  aws s3api head-bucket --bucket "$BUCKET_NAME" >/dev/null 2>&1 && echo "WARN: bucket aún existe" || echo "OK: bucket ya no existe (o no accesible)"
  ```
  {% include step_image.html %}
  ```bash
  aws sqs get-queue-url --queue-name "$QUEUE_NAME" --region "$AWS_REGION" >/dev/null 2>&1 && echo "WARN: queue aún existe" || echo "OK: queue ya no existe (o no accesible)"
  ```
  {% include step_image.html %}

#### Tarea 7.2

- {% include step_label.html %} Elimina los Providers (esto desinstala CRDs y controladores asociados).

  ```bash
  kubectl delete -f manifests/02-providers/provider-aws-s3.yaml --ignore-not-found
  ```
  ```bash
  kubectl delete -f manifests/02-providers/provider-aws-sqs.yaml --ignore-not-found
  ```

- {% include step_label.html %} Verifica que ya no existen providers instalados (puede tardar unos minutos).

  ```bash
  kubectl get providers || true
  ```
  ```bash
  kubectl get providerrevisions || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Desinstala Crossplane.

  ```bash
  helm uninstall crossplane -n crossplane-system || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el namespace de Crossplane (si quedó).

  ```bash
  kubectl delete ns crossplane-system --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si creaste el clúster solo para esta práctica, elimínalo con eksctl.

  > **NOTA:** El cluster tardara **9 minutos** aproximadamente en eliminarse.
  {: .lab-note .info .compact}

  ```bash
  eksctl delete cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica (si lo eliminaste) que ya no existe el clúster.

  ```bash
  aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" >/dev/null 2>&1 || echo "Cluster eliminado"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}