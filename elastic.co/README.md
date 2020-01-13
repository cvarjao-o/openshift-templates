# Elastic Search
## Generate Internal Certificates
For this step you are going to need 2 open terminals (Terminal 1, Terminal 2)
```
# From Terminal 1
oc run cert --image=docker.elastic.co/elasticsearch/elasticsearch:7.4.1 --command -it --rm --restart=Never -- bash

# From Terminal 2
oc rsh cert elasticsearch-certutil ca --out /tmp/elastic-stack-ca.p12 --pass ''
oc rsh cert elasticsearch-certutil cert --name elasticsearch-master --dns elasticsearch-master --ca /tmp/elastic-stack-ca.p12 --pass '' --ca-pass '' --out /tmp/elastic-certificates.p12
oc cp cert:/tmp/elastic-stack-ca.p12 ./
oc cp cert:/tmp/elastic-certificates.p12 ./
openssl pkcs12 -nodes -passin pass:'' -in elastic-certificates.p12 -out elastic-certificate.pem
openssl pkcs12 -clcerts -nokeys -passin pass:'' -in elastic-stack-ca.p12 -out elastic-ca.pem

oc delete pod/cert --force

oc create secret generic elastic-certificates --from-file=elastic-certificates.p12
oc create secret generic elastic-certificate-pem --from-file=elastic-certificate.pem
oc process -f elasticsearch-secrets.yaml  -p NAME=elastic-credentials | oc create -f -

# you may now fully exit/close from both terminals
```

## Create OpenShift Template
```
helm repo add elastic https://helm.elastic.co
helm fetch elastic/elasticsearch --version=7.4.1
helm template -f elasticsearch-values.yaml elasticsearch-7.4.1.tgz | oc create -f - --dry-run -o yaml > elasticsearch-deployment.yaml
```

## Deployment
```
oc process -f elasticsearch-deployment.yaml | oc create -f -
```

## Test
```
# from a pod terminal run:
curl -XGET -s -k --fail -u "${ELASTIC_USERNAME}:${ELASTIC_PASSWORD}" https://127.0.0.1:9200/

```

# KIBANA
```
helm fetch elastic/kibana --version=7.4.1
helm template kibana-7.4.1.tgz -f kibana-values.yaml | oc create -f - --dry-run -o yaml > kibana-deployment.yaml
oc process -f kibana-deployment.yaml | oc create -f - --dry-run

oc process -f kibana-secrets.yaml  -p NAME=kibana | oc create -f -

curl -k -u "${ELASTICSEARCH_USERNAME}:${ELASTICSEARCH_PASSWORD}" https://localhost:5601/app/kibana
```

# LOGSTASH
```
helm fetch elastic/logstash --version=7.4.1
oc run logstash --image=docker.elastic.co/logstash/logstash:7.4.1 -it --rm --restart=Never --command -- bash
```

# TODO
- auto-generate certificate private key/password instead of no password (empty).

# References:
- https://github.com/elastic/helm-charts/tree/master/elasticsearch
- https://github.com/elastic/helm-charts/tree/master/kibana
- https://github.com/elastic/helm-charts/tree/master/logstash
