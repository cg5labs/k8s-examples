# ElasticSearch and Kibana (ELK) 

## Requirements:
Default storage class defined for persistent storage claims, e.g. NFS.

## Install ELK Operator:

```
$ k apply -f elasticsearch/all-in-one.yaml
```

## Install ElasticSearch:

```
$ k create ns elastic
$ k apply -f elasticsearch/elk-mini.yaml -n elastic
```

## Install Kibana:

```
$ k apply -f elasticsearch/kibana.yaml -n elastic
```

## Check ELK cluster:
```
vagrant@cluster1-master1:~$ k get elastic -n elastic
NAME                                             HEALTH   NODES   VERSION   PHASE   AGE
elasticsearch.elasticsearch.k8s.elastic.co/elk   yellow   1       7.13.2    Ready   7h55m

NAME                               HEALTH   NODES   VERSION   AGE
kibana.kibana.k8s.elastic.co/elk   green    1       7.13.2    7h51m
vagrant@cluster1-master1:~$
```

## Access Kibana from Vagrant host:

```
vagrant@cluster1-master1:~$ k get secret -n elastic elk-es-elastic-user -o=yaml
vagrant@cluster1-master1:~$ kubectl port-forward --address 10.0.2.15 service/elk-kb-http 3000:5601
```

## ELK cluster customization
Config-as-code for index template:
```
$ k apply -f elasticsearch/es-customizer-cm.yaml
$ k apply -f elasticsearch/es-customizer-job.yaml
```
Dependencies: $ES_API_AUTH secret var from secret elk-es-elastic-user

## Kibana TLS with Traefik

The Argo CD application deploys `elk-instance/` into `monitoring`.
Traefik passes TLS through to Kibana, which presents the cert-manager certificate
from `es-tls` instead of an ECK-generated self-signed certificate.

```mermaid
flowchart LR
    Client[LAN client] -->|HTTPS es.cg5labs.net| Traefik
    Traefik -->|TLS passthrough, port 5601| Kibana
    CertManager[cert-manager] -->|Issues and renews| Secret[monitoring/es-tls]
    Secret -->|Certificate and private key| Kibana
```

- **Traefik:** Uses `traefik-external` and `websecure`, routes by SNI, and retains
  the `192.168.17.0/24` TCP allowlist. It does not terminate TLS or load `es-tls`.
- **Kibana:** References `spec.http.tls.certificate.secretName: es-tls`; the Secret
  must be in the same namespace and contain `tls.crt` and `tls.key`.
- **cert-manager:** The explicit `Certificate` in `es-cg5labs-net.yaml` replaces
  the old nginx Ingress's certificate issuance annotation and maintains `es-tls`.
  It requires the namespaced `Issuer` `monitoring/letsencrypt-production`.
  Keep that Issuer's ACME challenge solver functional independently of the
  retired nginx controller; cert-manager does not infer issuance from Traefik CRDs.

Sync the Argo CD application, or apply the manifests directly:

```sh
kubectl apply -f elasticsearch/elk-instance/es-cg5labs-net.yaml
kubectl wait -n monitoring --for=condition=Ready certificate/es-tls --timeout=180s
kubectl apply -n monitoring -f elasticsearch/elk-instance/kibana.yaml
kubectl rollout status -n monitoring deployment/elk-kb
curl -Iv https://es.cg5labs.net
```

Run the URL check from an allowed LAN address without `-k` so certificate trust
and hostname verification remain enabled. Do not reapply `es-ingress.yaml.nginx`.
