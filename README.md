***cert-manager required***

Create namespace for ldap
```bash
LDAPNS=ldap
kubectl create ns $LDAPNS
```

Generate key and cert for ldap ca
```bash
./bin/gen-ca
```

Deploy ldap
```bash
kustomize build . | kubectl -n ${LDAPNS} apply -f -
```

Drop ca cert into somewhere viya can consume it via `customer-provided-ca-certificates` generator (this is so Viya trusts LDAP)
```bash
cp ca/ldap_ca_cert.pem ../../viya/site-config/resources/ldap_ca.crt
```  

`sitedefault.yaml` too
```bash
cp resources/sitedefault.yaml ../../viya/site-config/resources
```

If you'd like to customize the directory... make changes to `bases/configmap.yaml`.  Then update + bounce ldap pod
```bash
kustomize build . | kubectl -n ${LDAPNS} apply -f -
kubectl delete pod ldap-server-xxxxxxxxxxxx # you'll need to specify your ldap pod 

```

test
```bash
./bin/test
```
