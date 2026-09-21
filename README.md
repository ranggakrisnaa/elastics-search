# Elasticsearch + Kibana (single node, VM)

## Jalanin di VM

```bash
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-es.conf

cp .env.example .env && $EDITOR .env
docker compose up -d
curl -u elastic:$ELASTIC_PASSWORD http://<private-ip>:9200
```

Kibana: `http://<private-ip>:5601` (login `elastic`).

## Firewall GCP

ES cuma pakai basic auth, tanpa TLS — jadi port 9200/5601 **wajib** cuma kebuka ke VPC:

```bash
gcloud compute firewall-rules create allow-es-from-gke \
  --network=<vpc> \
  --source-ranges=<gke-pod-cidr>,<gke-node-cidr> \
  --allow=tcp:9200,tcp:5601 \
  --target-tags=elasticsearch
```

VM-nya kasih network tag `elasticsearch`, dan jangan kasih external IP kalau nggak perlu.

## Nyambungin dari GKE

VM di VPC yang sama (atau peered) → app pakai private IP langsung.

```bash
kubectl create secret generic elastic-creds \
  --from-literal=url=http://<private-ip>:9200 \
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
  externalName: es.internal.example.com   # atau pakai Endpoints ke <private-ip>:9200
```

lalu app nembak `http://elasticsearch:9200`.
