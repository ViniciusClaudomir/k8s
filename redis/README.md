# Redis High Availability no Kubernetes (Minikube)

Cluster Redis com alta disponibilidade usando **Redis Sentinel** para monitoramento automático e failover.

---

## Arquitetura

```
┌─────────────────────────────────────────────────────────┐
│                    Namespace: redis-ha                   │
│                                                          │
│  ┌─────────────────┐       ┌─────────────────────────┐  │
│  │  redis-master-0 │◄──────│    redis-replica-0      │  │
│  │  (escrita/leitura)      │  (leitura / failover)   │  │
│  │   port: 6379    │       │      port: 6379          │  │
│  └────────┬────────┘       └────────────┬────────────┘  │
│           │                             │               │
│           │     monitora / failover     │               │
│           ▼                             ▼               │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Redis Sentinel                      │    │
│  │  sentinel-0 (port: 26379)                        │    │
│  │  sentinel-1 (port: 26379)                        │    │
│  │  quorum: 2 (ambos precisam concordar)            │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Componentes

| Componente           | Tipo        | Réplicas | Função                                     |
|----------------------|-------------|----------|--------------------------------------------|
| `redis-master`       | StatefulSet | 1        | Nó primário — aceita leitura e escrita     |
| `redis-replica`      | StatefulSet | 1        | Nó secundário — leitura e candidato a master |
| `redis-sentinel`     | StatefulSet | 2        | Monitora saúde e orquestra failover        |

### Serviços

| Serviço                | Tipo      | Porta  | Finalidade                          |
|------------------------|-----------|--------|-------------------------------------|
| `redis-master`         | Headless  | 6379   | DNS interno do StatefulSet          |
| `redis-master-svc`     | ClusterIP | 6379   | Acesso de clientes (escrita)        |
| `redis-replica`        | Headless  | 6379   | DNS interno do StatefulSet          |
| `redis-replica-svc`    | ClusterIP | 6379   | Acesso de clientes (leitura)        |
| `redis-sentinel`       | ClusterIP | 26379  | Consulta ao sentinel / descoberta   |

---

## Estrutura de Arquivos

```
redis/
├── 00-namespace.yaml          # Namespace redis-ha
├── master/
│   ├── 01-configmap.yaml      # Configuração do Redis Master
│   ├── 02-service-headless.yaml  # Service headless (DNS do pod)
│   ├── 03-service-clusterip.yaml # Service ClusterIP (acesso escrita)
│   └── 04-statefulset.yaml    # StatefulSet do Master
├── replica/
│   ├── 01-configmap.yaml      # Configuração da Replica
│   ├── 02-service-headless.yaml  # Service headless (DNS do pod)
│   ├── 03-service-clusterip.yaml # Service ClusterIP (acesso leitura)
│   └── 04-statefulset.yaml    # StatefulSet da Replica (com initContainer)
└── sentinel/
    ├── 01-configmap.yaml      # Configuração dos Sentinels
    ├── 02-service.yaml        # Service ClusterIP (porta 26379)
    └── 03-statefulset.yaml    # StatefulSet com 2 réplicas
```

---

## Pré-requisitos

- Minikube instalado e em execução
- `kubectl` configurado apontando para o Minikube

```bash
# Verificar se o Minikube está rodando
minikube status

# Iniciar se necessário
minikube start
```

---

## Deploy

### Opção 1 — Aplicar tudo de uma vez (recomendado)

```bash
kubectl apply -R -f redis/
```

> A flag `-R` aplica recursivamente todas as subpastas em ordem alfabética:
> `00-namespace` → `master/` → `replica/` → `sentinel/`
> Os initContainers nos pods de replica e sentinel garantem que eles aguardem o master estar pronto antes de inicializar.

### Opção 2 — Aplicar por componente

```bash
kubectl apply -f redis/00-namespace.yaml
kubectl apply -f redis/master/
kubectl apply -f redis/replica/
kubectl apply -f redis/sentinel/
```

> **Atenção:** `kubectl apply -f redis/` (sem `-R`) **não entra em subpastas** e só aplicaria o `00-namespace.yaml`.

---

## Verificação

### Checar status dos pods

```bash
kubectl get pods -n redis-ha -w
```

Saída esperada (todos `Running`):

```
NAME                READY   STATUS    RESTARTS   AGE
redis-master-0      1/1     Running   0          2m
redis-replica-0     1/1     Running   0          2m
redis-sentinel-0    1/1     Running   0          1m
redis-sentinel-1    1/1     Running   0          1m
```

### Checar serviços

```bash
kubectl get svc -n redis-ha
```

### Verificar replicação (master → replica)

```bash
# Entrar no master e verificar replicas conectadas
kubectl exec -it redis-master-0 -n redis-ha -- \
  redis-cli -a redis@HA2024 info replication
```

Saída esperada:
```
role:master
connected_slaves:1
slave0:ip=...,port=6379,state=online,offset=...,lag=0
```

### Verificar status dos Sentinels

```bash
# Consultar qual é o master atual
kubectl exec -it redis-sentinel-0 -n redis-ha -- \
  redis-cli -p 26379 sentinel get-master-addr-by-name mymaster

# Listar sentinels registrados
kubectl exec -it redis-sentinel-0 -n redis-ha -- \
  redis-cli -p 26379 sentinel sentinels mymaster

# Listar replicas conhecidas
kubectl exec -it redis-sentinel-0 -n redis-ha -- \
  redis-cli -p 26379 sentinel replicas mymaster
```

---

## Testando o Failover

### 1. Escrever uma chave no master

```bash
kubectl exec -it redis-master-0 -n redis-ha -- \
  redis-cli -a redis@HA2024 set chave-teste "valor-ha"
```

### 2. Confirmar que a replica recebeu a chave

```bash
kubectl exec -it redis-replica-0 -n redis-ha -- \
  redis-cli -a redis@HA2024 get chave-teste
```

### 3. Simular falha do master

```bash
kubectl delete pod redis-master-0 -n redis-ha
```

### 4. Observar o Sentinel promovendo a replica

```bash
# Acompanhar logs do sentinel em tempo real
kubectl logs -f redis-sentinel-0 -n redis-ha
```

Você verá mensagens como:
```
+sdown master mymaster ...
+odown master mymaster ... #quorum 2/2
+elected-leader master mymaster ...
+failover-state-send-slaveof-noone slave ...
+promoted-slave slave ...
+failover-end master mymaster ...
+switch-master mymaster ... (novo IP)
```

### 5. Verificar o novo master após failover

```bash
kubectl exec -it redis-sentinel-0 -n redis-ha -- \
  redis-cli -p 26379 sentinel get-master-addr-by-name mymaster
```

---

## Limpeza

```bash
# Remover todos os recursos do namespace
kubectl delete namespace redis-ha

# Remover PersistentVolumeClaims (não removidos com o namespace automaticamente no Minikube)
kubectl delete pvc -l app=redis -n redis-ha
```

---

## Configurações Importantes

### Senha

A senha padrão configurada é `redis@HA2024`. Para alterar:

1. Edite `requirepass` e `masterauth` nos ConfigMaps do master e replica
2. Edite `sentinel auth-pass` no ConfigMap do sentinel
3. Atualize as probes (`readinessProbe` / `livenessProbe`) nos StatefulSets

### Notas sobre Sentinel com 2 instâncias

Com 2 Sentinels e quorum configurado como `2`, **ambos** precisam estar disponíveis para executar um failover. Se um sentinel cair, o failover não ocorre.

> Em ambientes de **produção**, use **3 Sentinels** com quorum `2` (maioria). Assim, o cluster tolera a falha de 1 sentinel mantendo o quorum.

### DNS dos Pods (StatefulSet)

Os pods de um StatefulSet recebem DNS estáveis no formato:

```
<pod-name>.<service-headless>.<namespace>.svc.cluster.local
```

Exemplos:
- `redis-master-0.redis-master.redis-ha.svc.cluster.local`
- `redis-replica-0.redis-replica.redis-ha.svc.cluster.local`
- `redis-sentinel-0.redis-sentinel.redis-ha.svc.cluster.local`

---

## Conexão de Aplicações

Aplicações devem se conectar **via Sentinel**, não diretamente ao master, pois o endereço do master pode mudar após um failover.

Exemplo em Python com `redis-py`:

```python
from redis.sentinel import Sentinel

sentinel = Sentinel(
    [('redis-sentinel.redis-ha.svc.cluster.local', 26379)],
    socket_timeout=0.1,
    password='redis@HA2024'
)

master = sentinel.master_for('mymaster', socket_timeout=0.1)
slave  = sentinel.slave_for('mymaster', socket_timeout=0.1)

master.set('chave', 'valor')
print(slave.get('chave'))
```

---

## Storage em Produção (AWS)

Os `volumeClaimTemplates` dos StatefulSets usam a StorageClass padrão do cluster.
Para trocar o backend de storage, basta adicionar `storageClassName` nos arquivos
[master/04-statefulset.yaml](master/04-statefulset.yaml) e [replica/04-statefulset.yaml](replica/04-statefulset.yaml).

### EBS — Elastic Block Store (recomendado para Redis)

EBS usa protocolo de bloco, tem latência ~1ms e suporta `ReadWriteOnce` — ideal para Redis.

**Pré-requisito:** [EBS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) instalado no EKS.

```yaml
# Trecho a alterar em master/04-statefulset.yaml e replica/04-statefulset.yaml
volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3          # ou "gp2" se não tiver o EBS CSI Driver v2
      resources:
        requests:
          storage: 10Gi
```

StorageClasses disponíveis no EKS por padrão:

| Nome | Tipo | Notas |
|------|------|-------|
| `gp2` | EBS gp2 | padrão no EKS, disponível sem CSI Driver adicional |
| `gp3` | EBS gp3 | mais rápido e barato, requer EBS CSI Driver |

### EFS — Elastic File System (não recomendado para Redis)

EFS é NFS gerenciado com latência ~5-10ms. Suporta `ReadWriteMany`, mas Redis é
sensível a latência de disco — **use apenas se o requisito for compartilhamento de
volume entre múltiplos pods**, o que não se aplica ao Redis.

**Pré-requisito:** [EFS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html) + StorageClass criada manualmente.

```yaml
volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: ["ReadWriteMany"]
      storageClassName: efs-sc
      resources:
        requests:
          storage: 10Gi
```

### Comparativo

| Storage       | AccessMode | Latência  | Indicado para Redis |
|---------------|------------|-----------|---------------------|
| Minikube hostPath | RWO    | local     | apenas dev/local    |
| **EBS gp3**   | **RWO**    | **~1ms**  | **sim — produção**  |
| EBS gp2       | RWO        | ~1-2ms    | sim                 |
| EFS           | RWX        | ~5-10ms   | não recomendado     |

---

## Integração com FastAPI

A API deve se conectar ao Redis **via Service do Kubernetes**, usando o Sentinel
para descobrir dinamicamente qual pod é o master atual.

### Fluxo de comunicação

```
┌─────────────────┐        ┌──────────────────────────┐
│  Pod FastAPI     │──────►│  redis-sentinel (26379)   │
│  (qualquer ns)  │        │  "qual é o master agora?" │
└────────┬────────┘        └──────────────┬───────────┘
         │                                │ retorna IP:porta
         │         ┌──────────────────────┘
         ▼         ▼
┌─────────────────────────┐      ┌──────────────────────────┐
│  redis-master-svc:6379  │      │  redis-replica-svc:6379  │
│  (escrita)              │      │  (leitura)               │
└─────────────────────────┘      └──────────────────────────┘
```

### Variáveis de ambiente recomendadas

```yaml
# No Deployment da FastAPI
env:
  - name: REDIS_SENTINEL_HOST
    value: "redis-sentinel.redis-ha.svc.cluster.local"
  - name: REDIS_SENTINEL_PORT
    value: "26379"
  - name: REDIS_MASTER_NAME
    value: "mymaster"
  - name: REDIS_PASSWORD
    value: "redis@HA2024"   # em produção, use um Secret
```

### Código FastAPI com Sentinel

```python
# requirements.txt
# fastapi
# uvicorn
# redis[hiredis]

import os
from fastapi import FastAPI
from redis.sentinel import Sentinel

app = FastAPI()

sentinel = Sentinel(
    [(os.getenv("REDIS_SENTINEL_HOST"), int(os.getenv("REDIS_SENTINEL_PORT", 26379)))],
    socket_timeout=0.5,
    password=os.getenv("REDIS_PASSWORD"),
)

def get_master():
    return sentinel.master_for(
        os.getenv("REDIS_MASTER_NAME", "mymaster"),
        socket_timeout=0.5,
        password=os.getenv("REDIS_PASSWORD"),
    )

def get_replica():
    return sentinel.slave_for(
        os.getenv("REDIS_MASTER_NAME", "mymaster"),
        socket_timeout=0.5,
        password=os.getenv("REDIS_PASSWORD"),
    )


@app.post("/cache/{key}")
def set_key(key: str, value: str):
    get_master().set(key, value)
    return {"status": "ok", "key": key}

@app.get("/cache/{key}")
def get_key(key: str):
    value = get_replica().get(key)
    return {"key": key, "value": value}
```

> O Sentinel resolve automaticamente qual pod é o master após um failover.
> A aplicação não precisa ser reiniciada ou reconfigurada.
