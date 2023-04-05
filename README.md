# vault-helm-based-consul

## Install About
```
############ pre-set for data-backend : consul ########
#step-0 
  - consul helm-install for vault data-backend
    helm -n vault install consul hashicorp/consul -f consul-values.yaml
  OR
    helm -n vault upgrade -i consul hashicorp/consul -f consul-values.yaml

#step-1
  - telnet-test-pod setup
    > kubectl -n vault run -it --image nicolaka/netshoot testnet bash

  - checking consul service (consul-consul-server) with port 8500
    > telnet consul-consul-server 8500 
  OR
    > telnet consul-consul-server.vault.svc.cluster.local 8500

############ vault setup ############
#step-2
  - vault helm-install
    helm -n vault install vault hashicorp/vault -f vault-values.yaml
  OR
    helm -n vault upgrade -i vault hashicorp/vault -f vault-values.yaml
  
  - checking vault-pod
    kubectl get pods -l app.kubernetes.io/name=vault

############ vault Init-about ############
#step-3
  - 1. Initialising ( EACH-Vault-POD )
  - 2. Unsealing ( EACH-Vault-POD )
  - 3. Acquiring vault-cmd Permissions ( EACH-Vault-POD )
  - 4. Associating EACH-Vault-POD with K8S-AUTH ( EACH-Vault-POD )
  : 초기화 데이터의 경우 데이터 백엔드에 저장/유지되며 나머지 2~4의 경우, 볼트서버파드에 저장되어 노드 재기동시 2~4 번 과정을 다시 수행해야됨

  1. vault 서버 토큰키 확인 및 초기화 ( EACH-Vault-POD )
    kubectl -n vault exec vault-0 -- vault operator init \
      -key-shares=1 \
      -key-threshold=1 \
      -format=json > cluster-keys.json
    : 키쉐어 카운트 및 임계 카운트 설정 

  2. UNSEAL_KEY & ROOT_TOKEN 데이터 가공
    2-1. 키값 파싱
      cat cluster-keys.json | jq -r ".unseal_keys_b64[]" > ./vt-unseal-key.txt
      cat cluster-keys.json | jq -r ".root_token" > ./vt-root-key.txt

    2-2. 시크릿생성
      kubectl -n vault create secret generic vault-credential \
        --from-file=./vt-unseal-key.txt \
        --from-file=./vt-root-key.txt

    2-3. 시크릿 데이터 파싱 후 환경변수에 저장

      VAULT_UNSEAL_KEY=$(kubectl -n vault get secrets vault-credential -o jsonpath='{.data.vt-unseal-key\.txt}' | base64 -D)

      CLUSTER_ROOT_TOKEN=$(kubectl -n vault get secrets vault-credential -o jsonpath='{.data.vt-root-key\.txt}' | base64 -D)


  3. vault 상태확인 및 vault cmd 사용권한 획득 ( EACH-Vault-POD )
    3-1 상태확인
      > kubectl -n vault exec -it vault-0 -- vault status
      > kubectl -n vault exec -it vault-1 -- vault status

      : 초기화 및 씰 해재 상태확인, 씰 상태시 아래와 같이 unsealing
        > kubectl -n vault exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
        > kubectl -n vault exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY 

    3-2 vault cmd 사용권한 획득 
      : 루트 토큰으로 각 볼트파드 최초 1회 로그인 후 cli 권한 해제됨
        > kubectl -n vault exec vault-0 -- vault login $CLUSTER_ROOT_TOKEN
        > kubectl -n vault exec vault-1 -- vault login $CLUSTER_ROOT_TOKEN

  4. K8S-AUTH 연동 ( K8S 외부파드와 연동하기 위함 : EACH-Vault-POD )
    > kubectl exec -it vault-0 -- /bin/sh
    vault auth enable kubernetes

    vault write auth/kubernetes/config \
      kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
      token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
      kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

    > kubectl exec -it vault-1 -- /bin/sh 
    .......

    vault write auth/kubernetes/config \
      kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
      token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
      kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt 

```

## Key/Value SetUP on VAULT
```
#step-0 ( 시크릿 패스초기화 :Once Any vault-pod with data-backend : consul ) 
  > kubectl exec -it vault-0 -- /bin/sh
  vault secrets enable -path=secret kv-v2

#step-1 ( 키밸류 셋업 )
  vault kv put secret/test ID="test" password="passwd"

#step-2 ( 키밸류 확인 )
  > kubectl exec -it vault-1 -- /bin/sh
  vault kv get secret/test

```

## Using Key/Value from app-pod
```
### step-0 ( vault policy 생성 : vault-side 작업 ) 

> kubectl exec -it vault-1 -- /bin/sh

vault policy write secret-read - <<EOF
path "secret/*" {
  capabilities = ["read"]
}
EOF

#### step-1 ( vault role 생성 : vault-side 작업 ) 
# scret-read policy와 kubernetes interanl-app role 연결
# vault-test 네임스페이스에 vault-secret-read 서비스어카운트에게 권한부여

vault write auth/kubernetes/role/internal-app \
  bound_service_account_namespaces=vault \
  bound_service_account_names=vault \
  policies=secret-read \
  ttl=24h

### step-2 (app-pod 배포)

>>>>>  ex-yaml >>>>> 

apiVersion: v1
kind: Pod
metadata:
  name: tested-app-vault-ky-payroll
  namespace: vault
  labels:
    app: payroll
  annotations:
    vault.hashicorp.com/agent-inject: 'true'
    vault.hashicorp.com/role: 'internal-app'
    vault.hashicorp.com/agent-inject-secret-test: 'secret/test'
    vault.hashicorp.com/agent-inject-status: update
spec:
  serviceAccountName: vault
  containers:
    - name: payroll
      image: jweissig/app:0.0.1

#######################
=== 테스트앱파드 배포
k apply -f _ex-pod-injected-ky.yml


### step-3 (app-pod 에 삽입된 k/y 데이터 확인)

kubectl exec \
  $(kubectl get pod -l app=payroll -o jsonpath="{.items[0].metadata.name}" -n vault) \
  -c payroll -- cat /vault/secrets/test

```

## UnInstall About
```
helm -n vault uninstall vault

```