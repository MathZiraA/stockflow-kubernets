# StockFlow no Kubernetes

Microsserviços Java/Spring (gateway, pedidos, pagamentos) rodando no Kubernetes com descoberta de serviços (Eureka), banco persistente e escala automática.

| Problema | Solução | Prova |
|---|---|---|
| A API caía a cada deploy | Rolling update + readiness probe | No deploy, o pod antigo só saiu depois do novo ficar pronto |
| Perdia dados ao reiniciar | MySQL em StatefulSet + PVC | `mysql-0` apagado: voltou com o mesmo volume e os dados intactos |
| Não aguentava pico de acesso | HPA + Eureka | Sob carga, o pedidos escalou de 1 para 3 réplicas em ~15 s |

## Arquitetura

```
        gateway  (Deployment + Service ClusterIP :8082)
           │  consulta o Eureka
   ┌───────┼────────────┐
   ▼       ▼            ▼
pedidos ◄ pagamentos   eureka  (StatefulSet + headless :8081)
   │  (Deployment; pedidos com HPA 1–3)
   └───┬───┘
       ▼
     mysql  (StatefulSet + headless :3306 + PVC 1Gi)
```

- Todos os Services são `ClusterIP`: nada exposto para fora do cluster.
- Configuração fora da imagem: ConfigMap `dados` e Secret `senhas`.
- As imagens têm `localhost` fixo; funcionam no cluster porque variáveis de ambiente (`SPRING_DATASOURCE_URL`, `EUREKA_CLIENT_SERVICEURL_DEFAULTZONE`, `SERVER_PORT`) sobrescrevem o `application.properties`.
- Pedidos com readiness/liveness, `requests`/`limits` e heap do Java ajustado (`MaxRAMPercentage=60`).

## Como subir

Requisitos: cluster Kubernetes com a StorageClass `local-path` e o metrics-server.

```powershell
copy secret.example.yaml k8s\01-secret.yaml   # edite e troque TROQUE_AQUI
kubectl apply -f k8s/
kubectl port-forward svc/gateway 8082:8082 -n stockflow
```

Teste: http://localhost:8082/pedidos-ms/pedidos/porta

## Créditos

Código e imagens dos microsserviços: curso *Kubernetes com microsserviços* (Alura). Manifestos, adaptações e testes: Matheus Ziravello.
