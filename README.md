# Install

## Pre-reqs

1. enable privledged containers

2. increase ulimits on workers

    ```bash
    bosh ssh -d .... worker/... -c 'ulimit -n 1048576'
    ```

3. install heapster
    ```bash
    kubectl apply -f heapster
    ```

4. add a cluster role binding for the heapster service account

    ```bash
    kubectl create clusterrolebinding heapster --clusterrole cluster-admin --serviceaccount=kube-system:heapster
    ```

5. install helm https://docs.pivotal.io/runtimes/pks/1-4/helm.html

    ```bash
    kubectl apply -f helm/
    helm init --service-account tiller
    helm ls
    ```


6. create a lets encrypt cluster issuer good blog here on how to do this. https://blog.59s.io/cert-manager


    ```bash
    # create cert manager crds
    kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml
    ```

    ```bash
    # create a cert manager NS
    kubectl create namespace cert-manager
    kubectl label namespace cert-manager certmanager.k8s.io/disable-validation="true"
    ```

    ```bash
    # Add the Jetstack Helm repository
    helm repo add jetstack https://charts.jetstack.io

    # Update your local Helm chart repository cache
    helm repo update

    ## Install the cert-manager helm chart
    helm install \
    --name cert-manager \
    --namespace cert-manager \
    --version v0.7.0 \
    jetstack/cert-manager

    ```

7. create cluster issuer, this gets fiarly specific for GCP dns. the blog mentioned above covers AWS as well.

8. create a a gcp service account & k8s secret to store your GCP service account. require gcloud installed and logged in on your laptop

    ```bash
    chmod +x cert-manager/gcp-account.sh
    cert-manager/gcp-account.sh <gcp-project-name>
    ```

9.  update the cluster issuer, modify cert-manager/cluster-issuer.yml to add your email and gcp project name


    ```bash
    # apply the cluster issuer
    kubectl apply -f cert-manager/cluster-issuer.yml
    ```

## setup helm for eirini

1. add the eirini repo

    ```bash
    helm repo add eirini https://cloudfoundry-incubator.github.io/eirini-release
    ```

2. update the values file `helm/eirini-values.yml` for your domain and secrets

## Install UAA

3. install uaa via helm

    ```bash
    helm install --namespace uaa --name uaa --values helm/eirini-values.yaml eirini/uaa
    ```

2. get the UAA cert

    ```bash
    SECRET=$(kubectl get pods --namespace uaa -o jsonpath='{.items[?(.metadata.name=="uaa-0")].spec.containers[?(.name=="uaa")].env[?(.name=="INTERNAL_CA_CERT")].valueFrom.secretKeyRef.name}')
    CA_CERT="$(kubectl get secret $SECRET --namespace uaa -o jsonpath="{.data['internal-ca-cert']}" | base64 --decode -)"
    ```

3. get the LB ip for UAA and update your DNS

    ```bash
    kubectl get svc uaa-uaa-public
    ```
    ```bash
    *.uaa.app.$DOMAIN
    uaa.app.$DOMAIN
    ```

## Install CF

1. create a namespace for cf

    ```bash
    kubectl create namespace scf
    ```

2. create bit service certs using lets encypt, modify the file `cert-manager/bits-certs.yml` for your domains

    ```bash
    kubectl apply -f cert-manager/bits-certs.yml
    ```


3. export the BIT Certs

    ```bash
    BITS_TLS_KEY=$(kubectl get secret private-registry-cert --namespace scf -o jsonpath="{.data['tls\.key']}" | base64 --decode -)
    BITS_TLS_CRT=$(kubectl get secret private-registry-cert --namespace scf -o jsonpath="{.data['tls\.crt']}" | base64 --decode -)
    ```

4. install scf (currently there is a bug with some variables in the helm chart and we need to use the latest via the repo)

    ```bash
    git clone https://github.com/cloudfoundry-incubator/eirini-release.git

    cd eirini-release/helm/cf

    helm dependency update

    cd ../../../

    helm install eirini-release/helm/cf --namespace scf --name scf --values helm/eirini-values.yml --set "secrets.UAA_CA_CERT=${CA_CERT}" --set "eirini.secrets.BITS_TLS_KEY=${BITS_TLS_KEY}" --set "eirini.secrets.BITS_TLS_CRT=${BITS_TLS_CRT}" 

    ```



5. update dns

    ```bash
    kubectl get svc | grep Load

    ```
    ```bash
    bits: registry.app.$DOMAIN 
    router: *.app.$DOMAIN
    ssh: *.ssh.app.$DOMAIN , ssh.app.$DOMAIN
        
    tcp: *.tcp.app.$DOMAIN
    ```





6. login to cf

    ```bash
    cf api --skip-ssl-validation api.app.$DOMAIN
    cf login
    ```

7. create an org and space

    ```bash
    cf create-org eirini
    cf target -o eirini
    cf create-space eirini
    ```

8. push a sample app of some kind. NOTE: it looks like SCF has older buildpacks by default

9. view the k8s objects created behind the scenes

    ```bash
    kubectl get pods --namespace eirini
    ```

## References
https://github.com/cloudfoundry-incubator/eirini-release

https://github.com/paulczar/eirini-on-pks

