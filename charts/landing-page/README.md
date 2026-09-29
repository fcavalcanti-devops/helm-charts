# Landing Page
Landing Page to Apps/Tools

## Installing the Chart
```
helm repo add felipetech https://fcavalcanti-devops.github.io/helm-charts/felipetech
helm repo update
helm upgrade --install landing-page felipetech/landing-page --set application.ingress.hosts[0]=landing-page.domain.com
```

## Uninstalling the Chart
```
helm uninstall landing-page
```

## Global Parameters
| Chave | Tipo | Valor Padrão | Descrição |
|----------|----------|----------|----------|
| application.replicas | int | 1 | Número de réplicas do aplicativo. |
| application.image.name | string | felipecs8/landing-page | Nome da imagem Docker. |
| application.image.tag | string | v1 | Tag da imagem Docker. |
| application.service.type | string | ClusterIP | Tipo de serviço Kubernetes (ClusterIP, NodePort, LoadBalancer). |
| application.ingress.enabled | bool | true | Habilita o recurso de Ingress. |
| application.ingress.hosts | array | ["landing-page.127.0.0.1.nip.io"] | Lista de hosts para o Ingress. |

## GitHub Project
https://github.com/fcavalcanti-devops/landing-page