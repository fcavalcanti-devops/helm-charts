# kong-db-backed

Kong Gateway 3.9 OSS com Postgres e **sem** Ingress Controller, para usar o Kong Manager em modo escrita.

Este chart é de uso manual. Ele **não** tem Application no repositório `gitops` de propósito: não deve subir junto com a plataforma nem ficar rodando sozinho. Você instala quando precisa e desinstala quando termina.

## Por que existe

O gateway do dia a dia é DB-less e dirigido pelo Git: quem escreve a configuração é o Ingress Controller, a partir de `HTTPRoute` e `KongPlugin`. Nesse modo o Manager é somente leitura.

Aqui é o contrário. Sem controller, ninguém sobrescreve o que você criar pela UI, e a configuração vive no Postgres — como numa instalação clássica do Kong. Em troca, nada aqui vem do Git: `HTTPRoute` e `KongPlugin` não têm efeito nesta stack.

## Instalar

A senha do banco não está no Git. Crie o Secret antes:

```bash
kubectl create namespace kong-db-backed
kubectl -n kong-db-backed create secret generic kong-postgres --from-literal=password='SUA-SENHA'
```

```bash
helm dependency build charts/kong-db-backed
helm install kong-db-backed charts/kong-db-backed -n kong-db-backed
```

O primeiro start demora: o job de migração só cria o schema depois que o Postgres aceita conexão.

Instalar em outro namespace exige ajustar `kong.env.pg_host`, que carrega o namespace no nome DNS.

## Acessar o Manager

Todos os Services são `ClusterIP`. A Admin API do Kong OSS não tem autenticação e o Manager OSS não tem login — por isso nada aqui é publicado na rede.

```bash
kubectl port-forward -n kong-db-backed svc/kong-db-backed-kong-admin 8001:8001
kubectl port-forward -n kong-db-backed svc/kong-db-backed-kong-manager 8002:8002
```

Abra http://localhost:8002. São necessários os dois: a 8002 serve a interface e a 8001 é a API que ela consulta. Como a Admin API é HTTP puro, não há certificado autoassinado para aceitar antes.

Para testar com `curl` as rotas que você criar pela UI:

```bash
kubectl port-forward -n kong-db-backed svc/kong-db-backed-kong-proxy 8000:8000
```

Se mudar as portas do port-forward, ajuste `kong.env.admin_gui_url` e `kong.env.admin_gui_api_url` nos values: são endereços que o **navegador** resolve, não o cluster.

## Remover

```bash
helm uninstall kong-db-backed -n kong-db-backed
kubectl delete namespace kong-db-backed
```

Apagar o namespace leva junto o PVC do Postgres e o Secret da senha. O gateway em `kong-system` não é afetado.

## Values principais

| Chave | Padrão | Para que serve |
|---|---|---|
| `postgres.image` | `postgres:16.15` | Linha 16 é a testada com o Kong 3.9 |
| `postgres.storage` | `5Gi` | Tamanho do PVC |
| `postgres.existingSecret` | `kong-postgres` / `password` | Secret criado à mão com a senha |
| `kong.ingressController.enabled` | `false` | É esta flag que devolve a escrita ao Manager |
| `kong.env.pg_host` | `kong-postgres.kong-db-backed.svc.cluster.local` | Precisa bater com o namespace da instalação |
