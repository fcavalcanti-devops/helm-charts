# Guia: Kong com banco e o conversor

Passo a passo do `kong-db-backed` com o app conversor. Os IPs deste laboratório são o master `192.168.2.217` e o worker `192.168.2.196`. Se o node mudar de IP, troque só o número.

Este Kong não tem Ingress Controller. `HTTPRoute` e `KongPlugin` não entram aqui. A rota nasce no Manager e fica no Postgres. O `HTTPRoute` do chart do conversor serve o outro gateway, o `kong` do namespace `kong`, que é DB-less e dirigido pelo Git.

## 1. Apagar uma instalação anterior

Se o conversor veio de uma Application do Argo CD, apague essa Application antes. Senão o Argo recria o Service.

```bash
helm uninstall teste -n default
kubectl delete deploy,svc -n default -l app=teste-conversor-temperatura
helm uninstall kong-db-backed -n kong-db-backed
kubectl delete namespace kong-db-backed
```

## 2. Instalar o Kong

A senha do banco não fica no Git. Rode na máquina onde está este chart.

```bash
kubectl create namespace kong-db-backed
kubectl -n kong-db-backed create secret generic kong-postgres --from-literal=password='SUA-SENHA'
helm dependency build charts/kong-db-backed
helm install kong-db-backed charts/kong-db-backed -n kong-db-backed
```

Espere o Postgres e o Kong ficarem `Running`. O primeiro start demora por causa da migração do banco.

```bash
kubectl get pods -n kong-db-backed
kubectl get svc -n kong-db-backed kong-db-backed-kong-proxy
```

O proxy é `LoadBalancer` na porta 80. No k3s o ServiceLB publica essa porta em todos os nodes, então a URL do app não leva porta. O `EXTERNAL-IP` lista os IPs dos nodes. A Admin API (`8001`) e o Manager (`8002`) ficam `ClusterIP`: a Admin API do Kong OSS não tem autenticação e o Manager OSS não tem login.

## 3. Instalar o conversor

Sem `HTTPRoute`. Use uma tag que exista no Docker Hub. A tag `b416c3cf83a20d7c006b46a10673b0e571e53989` não existe e o Pod fica em `ImagePullBackOff`. Com o Pod parado, o Kong responde `An invalid response was received from the upstream server`.

```bash
helm repo add felipetech https://fcavalcanti-devops.github.io/helm-charts/felipetech
helm repo update
helm upgrade --install teste felipetech/conversor-temperatura -n default --set application.image.tag=latest
kubectl get pods,svc -n default
```

O release `teste` gera o Service `teste-conversor-temperatura` no namespace `default`, porta 80, encaminhando para o container na 8080. O nome é `{release}-conversor-temperatura`.

## 4. Abrir o Manager

`kong.env.admin_gui_url` e `kong.env.admin_gui_api_url` são os endereços que o navegador resolve, não o cluster.

Na mesma máquina do port-forward, o padrão `localhost` vale. São necessários os dois forwards, ao mesmo tempo. Abra exatamente `http://localhost:8002`. `127.0.0.1` deixa a tela vazia, porque a Admin API só libera a origem gravada em `admin_gui_url`.

```bash
kubectl port-forward -n kong-db-backed svc/kong-db-backed-kong-admin 8001:8001
kubectl port-forward -n kong-db-backed svc/kong-db-backed-kong-manager 8002:8002
```

Para abrir o Manager de outra máquina, fora do cluster, troque os dois valores pelo IP de um node e rode `helm upgrade` antes de usar a UI:

```yaml
kong:
  env:
    admin_gui_url: http://192.168.2.217:8002
    admin_gui_api_url: http://192.168.2.217:8001
```

A página fica `http://192.168.2.217:8002`. As portas 8001 e 8002 têm que responder nesse IP. No padrão elas são ClusterIP, então publique o Admin e o Manager (NodePort ou LoadBalancer). Sem isso o navegador não alcança a API e os campos da UI ficam em `--`.

## 5. Criar o Service e a Route

Em **Gateway Services**, **New Gateway Service**:

| Campo | Valor |
|---|---|
| Name | `teste` |
| Protocol | `http` |
| Host | `teste-conversor-temperatura.default.svc.cluster.local` |
| Port | `80` |
| Path | `/` |

Dentro dele, **Add Route**:

| Campo | Valor |
|---|---|
| Name | `teste` |
| Protocols | `http` e `https` |
| Hosts | `conversor.192.168.2.217.nip.io` |
| Paths | `/` |
| Strip Path | desligado |

O host da route é o nome que o cliente envia. O nip.io só resolve para o IP do node. Quem decide o upstream é o Kong, pelo header `Host`.

## 6. Testar em HTTP

```bash
curl -H "Host: conversor.192.168.2.217.nip.io" http://192.168.2.217/health
```

A resposta é `OK`. No navegador, na mesma rede: `http://conversor.192.168.2.217.nip.io/`.

O endereço sem porta só funciona porque o proxy escuta na 80. NodePort fica na faixa 30000–32767 e a porta aparece na URL.

## 7. HTTPS para um subdomínio

Deixar o TLS do proxy ligado é o estado normal. A porta 80 continua aberta e a 443 passa a atender HTTPS. Cada certificado de subdomínio fica no Manager, no Postgres, e não entra no `values.yaml`.

O Kong só sobe com a escuta SSL se já existir algum certificado: um Secret montado no pod, ou um certificado criado no Manager. Sem nenhum dos dois, o pod não inicia a porta 443.

Gere um certificado só para o host da route. O IP é privado, então um Let's Encrypt público não emite para `conversor.192.168.2.217.nip.io`.

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=conversor.192.168.2.217.nip.io" \
  -addext "subjectAltName=DNS:conversor.192.168.2.217.nip.io"
```

No Manager, em **Certificates**, cole `tls.crt` e `tls.key`. No SNI, coloque só `conversor.192.168.2.217.nip.io`. Outro host não usa esse certificado.

Ligue a porta 443 uma vez, no `values.yaml`, e aplique:

```yaml
kong:
  proxy:
    tls:
      enabled: true
      servicePort: 443
      containerPort: 8443
```

```bash
helm upgrade kong-db-backed charts/kong-db-backed -n kong-db-backed
```

Crie o certificado no Manager antes desse upgrade, ou monte um Secret default e aponte o Kong para ele:

```bash
kubectl -n kong-db-backed create secret tls kong-proxy-tls --cert=tls.crt --key=tls.key
```

```yaml
kong:
  secretVolumes:
    - kong-proxy-tls
  env:
    ssl_cert: /etc/secrets/kong-proxy-tls/tls.crt
    ssl_cert_key: /etc/secrets/kong-proxy-tls/tls.key
```

O Secret default vale como certificado de fallback. O certificado de um subdomínio específico continua sendo o do SNI no Manager.

Abra `https://conversor.192.168.2.217.nip.io/`. O navegador avisa que o certificado não é de uma autoridade pública. Aceite a exceção nessa máquina. O `/health` em HTTPS responde `OK` com o Pod do conversor em `Running`.

## 8. Remover

```bash
helm uninstall teste -n default
helm uninstall kong-db-backed -n kong-db-backed
kubectl delete namespace kong-db-backed
```

Apagar o namespace leva o PVC do Postgres e o Secret da senha. O gateway do namespace `kong` não é afetado.
