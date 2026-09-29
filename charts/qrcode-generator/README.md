# QRCode.js
Generate QR Codes in a simplified way

## Installing the Chart
```
helm repo add felipetech https://fcavalcanti-devops.github.io/helm-charts/felipetech
helm repo update
helm upgrade --install qrcode-generator felipetech/qrcode-generator --set application.ingress.hosts[0]=qrcode.domain.com
```

### Rodando Localmente

```
helm upgrade --install qrcode-generator felipetech/qrcode-generator --set application.ingress.hosts[0]=qrcode.domain.com
```

### Rodando através da ACM da AWS

Configurar CIDR Block VPC e arn ACM no arquivo deploy-ingress-nginx-eks.yaml
kubectl apply -f deploy-ingress-nginx-eks.yaml.yaml

```
helm upgrade --install web  . --set application.ingress.hosts[0]=teste-qrcode.lab.felipedevops.com --set application.ingress.ingressClassName=nginx --set application.ingress.sslRedirect.enabled=true
```

### Rodando através do TLS com cert gerado pelo cert-manager (LetsEncrypt)

```
helm upgrade --install web  . --set application.ingress.hosts[0]=teste-qrcode.lab.felipedevops.com --set application.ingress.ingressClassName=nginx --set application.ingress.tls.enabled=true
```

## Uninstalling the Chart
```
helm uninstall qrcode-generator
```

## Global Parameters
| Chave | Tipo | Valor Padrão | Descrição |
|----------|----------|----------|----------|
| application.replicas | int | 1 | Número de réplicas do aplicativo. |
| application.image.name | string | felipecs8/qrcode-generator | Nome da imagem Docker. |
| application.image.tag | string | v1 | Tag da imagem Docker. |
| application.service.type | string | ClusterIP | Tipo de serviço Kubernetes (ClusterIP, NodePort, LoadBalancer). |
| application.ingress.enabled | bool | true | Habilita o recurso de Ingress. |
| application.ingress.hosts | array | ["qrcode.127.0.0.1.nip.io"] | Lista de hosts para o Ingress. |

## GitHub Project
https://github.com/fcavalcanti-devops/qrcode-generator