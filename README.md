# FindFace Helm Chart - Manual de Deploy

Este chart implementa o recorte analisado do ambiente exportado:

- `findface-extraction-api`
- `findface-video-worker` (apenas 1 instancia)

Dependencias externas permanecem fora do chart:

- NTLS (`externalDependencies.ntlsAddress`)
- Video Manager RPC (`externalDependencies.videoManagerRpcAddress`)
- Video Storage URL (`externalDependencies.videoStorageUrl`)

## Estrutura do chart

```text
charts/findface/
  Chart.yaml
  values.yaml
  templates/
    extraction-api/
      configmap.yaml
      pvc-cache.yaml
      deployment.yaml
      service.yaml
      route.yaml
    video-worker/
      configmap.yaml
      pvc-cache.yaml
      pvc-recorder.yaml
      deployment.yaml
      service.yaml
    pvc-models.yaml
    NOTES.txt
```

## Pre-requisitos

- Kubernetes 1.24+ ou OpenShift 4.x
- Helm 3.x
- Classe de storage disponivel para PVCs
- Acesso de rede dos pods para:
  - NTLS
  - Video Manager RPC
  - Video Storage
- Se usar GPU:
  - Device plugin NVIDIA ativo no cluster
  - Nodes com GPU e rotulo/afinidade conforme sua politica
- Se imagens privadas exigirem autenticacao:
  - Secret de pull criado no namespace

## Valores principais (values)

Arquivo padrao: `charts/findface/values.yaml`

Campos mais importantes:

- `externalDependencies.ntlsAddress`
- `externalDependencies.videoManagerRpcAddress`
- `externalDependencies.videoStorageUrl`
- `imagePullSecrets`
- `models.pvc.*`
- `extractionApi.*`
- `videoWorker.*`
- `route.extractionApi.*` (OpenShift)

## Exemplo de values para ambiente

Crie um arquivo `values-prod.yaml`:

```yaml
imagePullSecrets:
  - regcred

externalDependencies:
  ntlsAddress: "10.32.200.19:3133"
  videoManagerRpcAddress: "10.32.200.19:18811"
  videoStorageUrl: "http://video-storage.svc.cluster.local:18611"

models:
  pvc:
    storageClassName: "ocs-storagecluster-ceph-rbd"
    size: 20Gi

extractionApi:
  gpu:
    enabled: true
    count: 1
  cache:
    pvc:
      storageClassName: "ocs-storagecluster-ceph-rbd"
      size: 5Gi

videoWorker:
  gpu:
    enabled: true
    count: 1
  cache:
    pvc:
      storageClassName: "ocs-storagecluster-ceph-rbd"
      size: 5Gi
  recorder:
    pvc:
      storageClassName: "ocs-storagecluster-ceph-rbd"
      size: 20Gi
```

## Deploy (OpenShift)

### 1) Criar projeto/namespace

```bash
oc new-project findface
```

Ou, em Kubernetes:

```bash
kubectl create namespace findface
```

### 2) (Opcional) criar pull secret para registry privado

```bash
oc -n findface create secret docker-registry regcred \
  --docker-server=docker.int.ntl \
  --docker-username='<usuario>' \
  --docker-password='<senha>' \
  --docker-email='<email>'
```

### 3) Validar chart antes de aplicar

```bash
helm lint ./charts/findface
helm template findface ./charts/findface -n findface -f values-prod.yaml
```

### 4) Instalar/atualizar release

```bash
helm upgrade --install findface ./charts/findface \
  -n findface \
  -f values-prod.yaml
```

### 5) (Opcional) habilitar Route para extraction-api

No `values-prod.yaml`:

```yaml
route:
  extractionApi:
    enabled: true
    host: "extraction-api.apps.seu-cluster.exemplo"
```

Ou por linha de comando:

```bash
helm upgrade --install findface ./charts/findface \
  -n findface \
  -f values-prod.yaml \
  --set route.extractionApi.enabled=true \
  --set route.extractionApi.host=extraction-api.apps.seu-cluster.exemplo
```

## Verificacao pos-deploy

```bash
oc -n findface get pods
oc -n findface get pvc
oc -n findface get svc
oc -n findface get route
```

Conferir manifests renderizados do release aplicado:

```bash
helm get manifest findface -n findface
```

## Upgrade, rollback e uninstall

Upgrade:

```bash
helm upgrade findface ./charts/findface -n findface -f values-prod.yaml
```

Historico:

```bash
helm history findface -n findface
```

Rollback:

```bash
helm rollback findface <REVISAO> -n findface
```

Uninstall:

```bash
helm uninstall findface -n findface
```

## Troubleshooting rapido

- Pods nao sobem por imagem:
  - Verifique `imagePullSecrets` e credenciais de registry.
- Pods nao sobem por GPU:
  - Verifique device plugin NVIDIA e capacidade de nodes.
- Falha de conexao com servicos externos:
  - Revise `externalDependencies.*` e politicas de rede.
- PVC pendente:
  - Revise `storageClassName`, quotas e capacidade do cluster.
- Necessidade de `hostNetwork`:
  - O padrao esta `false`; habilite somente se validado no seu ambiente.

## Observacoes de escopo

- O chart foi simplificado para **um unico video-worker**.
- A topologia completa FindFace Multi nao esta integralmente modelada aqui.
- Antes de producao, valide dependencias externas, GPU e storage conforme sua infraestrutura.
