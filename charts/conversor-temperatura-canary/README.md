# Conversor Temperatura — Canary (teste)

Chart **apenas para testes** com [Argo Rollouts](https://argo-rollouts.readthedocs.io/).  
Produção/homolog “normal” continua no chart `conversor-temperatura` (Deployment).

## Pré-requisito

Controller Argo Rollouts no cluster:

```bash
kubectl apply --server-side --force-conflicts -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

## Install (local / lab)

```bash
helm upgrade --install conversor-canary ./charts/conversor-temperatura-canary \
  -n conversor-canary --create-namespace \
  --set application.image.tag=<sha-ou-latest>
```

## Canary

Steps padrão: `50%` → pause `60s` → `100%` (`application.canary.steps`).

```bash
kubectl get rollout -n conversor-canary -w
kubectl argo rollouts get rollout conversor-canary-conversor-temperatura -n conversor-canary -w
```

Testar tráfego (port-forward gruda num pod; use curl in-cluster):

```bash
kubectl port-forward -n conversor-canary svc/conversor-canary-conversor-temperatura 8080:80
```

## Argo CD

Aponte uma Application para `charts/conversor-temperatura-canary` (namespace de teste, ex.: `conversor-canary`), separado da app homolog/prod.
