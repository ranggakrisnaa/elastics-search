# Elasticsearch + Kibana (single node, VM)

## Jalanin di VM

```bash
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-es.conf

cp .env.example .env && $EDITOR .env
docker compose up -d
curl -u elastic:$ELASTIC_PASSWORD http://<private-ip>:$ES_PORT
```

Kibana: `http://<private-ip>:$KIBANA_PORT` (login `elastic`).

Port host diambil dari allowlist firewall (lihat `ES_PORT` / `KIBANA_PORT` di `.env`), port di dalam container tetap 9200/5601.

## Firewall GCP

ES cuma pakai basic auth, tanpa TLS — jadi port 9200/5601 **wajib** cuma kebuka ke VPC:

```bash
gcloud compute firewall-rules create allow-es-from-gke \
  --network=<vpc> \
  --source-ranges=<gke-pod-cidr>,<gke-node-cidr> \
  --allow=tcp:8030,tcp:8040 \
  --target-tags=elasticsearch
```

VM-nya kasih network tag `elasticsearch`, dan jangan kasih external IP kalau nggak perlu.

## Nyambungin dari GKE

VM di VPC yang sama (atau peered) → app pakai private IP langsung.

```bash
kubectl create secret generic elastic-creds \
  --from-literal=url=http://<private-ip>:8030 \
  --from-literal=username=elastic \
  --from-literal=password=<ELASTIC_PASSWORD>
```

Biar nggak hardcode IP di app, bikin Service tanpa selector:

```yaml
apiVersion: v1
kind: Service
metadata: { name: elasticsearch }
spec:
  type: ExternalName
  externalName: es.internal.example.com   # atau pakai Endpoints ke <private-ip>:8030
```

lalu app nembak `http://elasticsearch:8030`.
