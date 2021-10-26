# How to Deploy Basic Openldap in K8S

## 1. Create namespace for ldap
```bash
LDAPNS=ldap
kubectl create ns $LDAPNS
```

## 2. Generate cert for ldap (requires openssl)
```bash
./bin/gen-cert
```

## 3. (Optional) Use a private registry

By default, this deployment will pull and run a public openldap image (osixia/openldap).  If your cluster is able to pull images from public internet (dockerhub specifically), then skip this section.  If you want to use your own version (i.e., your cluster does not have access to docker hub), then there are one or three things you must do before moving on:

#### 3.1 - push your image to your registry

There is a openldap.tgz file here, which is a compressed archive of the [osixia/openldap image](https://github.com/osixia/docker-openldap).   you can unpack this file (`tar xzfv openldap.tgz`), load the image to your local docker (`docker load --input openldap.tar`), tag the image to your repo (`docker tag openldap my-registry.com/my-ldap:v1`), and push the image to your registry (`docker push my-registry.com/my-ldap:v1`).  

You can also visit the osixia github and build this image yourself following their instructions.  In any event, we will ultimately need an openldap image in your registry.  

Once this is done, you will uncomment & modify the image transfomer in the `kustomization.yaml` to point to your version.  

#### 3.2 - (optional - only if your cluster does not have full read access to your register) create imagePullSecret which will authorize access to your registry

Generate a local `.dockerconfigjson` file (update these variables according to your environment):
```bash
REGISTRYURL=
MIRRORUSERNAME=
MIRRORPASSWORD= 

kubectl create -n $LDAPNS secret docker-registry --dry-run=client regcred \
--docker-server=$REGISTRYURL \
--docker-username=$MIRRORUSERNAME \
--docker-password=$MIRRORPASSWORD \
--output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode > deploy/.dockerconfigjson
```

#### 3.3 - (optional - only if your cluster does not have full read access to your register) make use of this `.dockerconfigjson` file by simply un-commenting the two relevant lines in `kustomization.yaml`
```
# if your openldap image is coming from a private registry, you will also need to uncomment these lines to add this imagePullSecret (instructions for creating imagePullSecret are in the README)
#generators:
#  - image-pull-secret.yaml

#transformers:
#  - add-image-pull-secret.yaml
```

## 4. Deploy ldap
```bash
kustomize build deploy/ | kubectl -n ${LDAPNS} apply -f -
```

## 5. Use ldap
Drop the ldap cert somewhere viya can consume it via `customer-provided-ca-certificates` generator (this is so Viya trusts LDAP)
```bash
cp resources/ldap_cert.pem ../../viya/site-config/resources/ldap_ca.crt
```  

`sitedefault.yaml` too
```bash
cp resources/sitedefault.yaml ../../viya/site-config/resources
```

## Appendix
If you'd like to customize the directory... make changes to `directory.yaml`.  Then update + bounce ldap pod
```bash
kustomize build . | kubectl -n ${LDAPNS} apply -f -
kubectl delete pod ldap-server-xxxxxxxxxxxx # you'll need to specify your ldap pod 
```

test
```bash
./bin/test
```
