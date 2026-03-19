# All-in-One Secure Vault Testing Guide

This document captures the local test flow used for the `all-in-one` chart changes around:

- symmetric encryption key support
- secure-vault local secret mounting
- asymmetric backward-compatibility checks
- image override testing with a custom Docker Hub image

It is intended as a future reference so the same scenarios can be rerun without rebuilding the steps from scratch.

## What Was Tested

The following scenarios were prepared:

1. Symmetric encryption with no key
2. Symmetric encryption with a plain text internal encryption key
3. Symmetric secure-vault with a local mounted master key and a plain text internal encryption key
4. Symmetric secure-vault negative test with pre-encrypted values
5. Symmetric secure-vault negative test with the wrong master key
6. Asymmetric secure-vault backward-compatibility test

## Important Findings

Current results from testing:

- Scenario 1: expected Helm failure, and it passed as a negative test
- Scenario 2: passed
- Scenario 3: local secret mount worked, but `cipher-text.properties` remained plain text, so the secure-vault symmetric path still needs ciphertext generation before startup
- Scenario 6: the `m2` keystore path issue was chart-side and was fixed by making the runtime entrypoint rewrite the keystore path in `secret-conf.properties`

Current known gap:

- secure-vault startup still expects `repository/conf/security/cipher-text.properties` to contain encrypted values
- in the tested flow, that file remained plain text
- because of that, secure-vault symmetric startup is still blocked until ciphertext generation is introduced before server startup

## Files Used During Testing

Key override files:

- `all-in-one/test-overrides/00-disable-gateway-api.yaml`
- `all-in-one/test-overrides/00-image-prealpha-20260319-1.yaml`
- `all-in-one/test-overrides/01-negative-no-encryption-key.yaml`
- `all-in-one/test-overrides/02-plaintext-internal-encryption-key.yaml`
- `all-in-one/test-overrides/03-secure-vault-local-master-plain-internal-key.yaml`
- `all-in-one/test-overrides/04-negative-secure-vault-preencrypted-inputs.yaml`
- `all-in-one/test-overrides/05-negative-secure-vault-wrong-master-key.yaml`
- `all-in-one/test-overrides/06-asymmetric-backward-compatibility.yaml`

Local secret files:

- `all-in-one/test-overrides/secrets/local-secure-vault-good.yaml`
- `all-in-one/test-overrides/secrets/local-secure-vault-wrong-master.yaml`

## Image Used

The tested image was:

```text
docker.io/aqeeltm/wso2am:4.7.0-prealpha-20260319-1
sha256:8dd7bcabdc3143cf050f1e0f2a9f2f1b9a2cfa1dcaf0fa4cfbc66231d17dce67
```

If a new image is needed later:

```bash
export IMAGE_TAG=4.7.0-prealpha-YYYYMMDD-N
export IMAGE=aqeeltm/wso2am:${IMAGE_TAG}

docker build -t "${IMAGE}" .
docker push "${IMAGE}"
docker buildx imagetools inspect "${IMAGE}"
```

Then update `all-in-one/test-overrides/00-image-prealpha-20260319-1.yaml` with the new tag and digest.

If only Helm templates/configmaps changed, no image rebuild is needed.

## Install Ingress

Install once:

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

## Clean Slate Commands

Remove any previous test releases:

```bash
helm uninstall apim-no-key -n apim-no-key || true
helm uninstall apim-plain -n apim-plain || true
helm uninstall apim-sv-local -n apim-sv-good || true
helm uninstall apim-sv-preenc -n apim-sv-negative || true
helm uninstall apim-sv-wrong -n apim-sv-wrong-master || true
helm uninstall apim-asym -n apim-asym || true
```

Delete previous namespaces:

```bash
kubectl delete ns apim-no-key --ignore-not-found=true
kubectl delete ns apim-plain --ignore-not-found=true
kubectl delete ns apim-sv-good --ignore-not-found=true
kubectl delete ns apim-sv-negative --ignore-not-found=true
kubectl delete ns apim-sv-wrong-master --ignore-not-found=true
kubectl delete ns apim-asym --ignore-not-found=true
```

Check they are gone:

```bash
kubectl get ns | egrep 'apim-no-key|apim-plain|apim-sv-good|apim-sv-negative|apim-sv-wrong-master|apim-asym' || true
```

## Scenario 1: Symmetric Without Encryption Key

Expected result: Helm should fail with a validation error.

```bash
helm upgrade --install apim-no-key ./all-in-one \
  -n apim-no-key --create-namespace \
  -f all-in-one/default_values.yaml \
  -f all-in-one/test-overrides/00-disable-gateway-api.yaml \
  -f all-in-one/test-overrides/00-image-prealpha-20260319-1.yaml \
  -f all-in-one/test-overrides/01-negative-no-encryption-key.yaml
```

Expected failure:

```text
Encryption key is required for symmetric encryption...
```

## Scenario 2: Symmetric With Plain Text Internal Encryption Key

```bash
helm upgrade --install apim-plain ./all-in-one \
  -n apim-plain --create-namespace \
  -f all-in-one/default_values.yaml \
  -f all-in-one/test-overrides/00-disable-gateway-api.yaml \
  -f all-in-one/test-overrides/00-image-prealpha-20260319-1.yaml \
  -f all-in-one/test-overrides/02-plaintext-internal-encryption-key.yaml
```

Verify:

```bash
kubectl get pods -n apim-plain
kubectl get deploy -n apim-plain
kubectl logs -n apim-plain deploy/apim-plain-wso2am-all-in-one-am-deployment-1 --tail=200
```

Observed result:

- chart installed successfully
- pod became ready
- server startup completed
- this scenario passed

Cleanup:

```bash
helm uninstall apim-plain -n apim-plain
kubectl delete ns apim-plain
```

## Scenario 3: Symmetric Secure Vault With Local Mounted Master Key

Create namespace and secret:

```bash
kubectl create ns apim-sv-good
kubectl apply -n apim-sv-good -f all-in-one/test-overrides/secrets/local-secure-vault-good.yaml
```

Install:

```bash
helm upgrade --install apim-sv-local ./all-in-one \
  -n apim-sv-good \
  -f all-in-one/default_values.yaml \
  -f all-in-one/test-overrides/00-disable-gateway-api.yaml \
  -f all-in-one/test-overrides/00-image-prealpha-20260319-1.yaml \
  -f all-in-one/test-overrides/03-secure-vault-local-master-plain-internal-key.yaml
```

Verify pod and logs:

```bash
kubectl get pods -n apim-sv-good
kubectl get deploy -n apim-sv-good
kubectl logs -n apim-sv-good deploy/apim-sv-local-wso2am-all-in-one-am-deployment-1 --since=10m
```

Useful inspection commands:

```bash
POD=$(kubectl get pod -n apim-sv-good -l node=apim-sv-local-wso2am-all-in-one-am-1 -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n apim-sv-good -it "$POD" -- sh -c 'cat /mnt/local-secrets/masterEncryptionKey'
kubectl exec -n apim-sv-good -it "$POD" -- sh -c 'find /home/wso2carbon -name "encryption-key-persist" 2>/dev/null'
kubectl exec -n apim-sv-good -it "$POD" -- sh -c 'sed -n "1,120p" /home/wso2carbon/wso2am-4.7.0-m2/repository/conf/security/cipher-text.properties'
kubectl exec -n apim-sv-good -it "$POD" -- sh -c 'sed -n "1,140p" /home/wso2carbon/docker-entrypoint.sh'
```

Observed result:

- mounted master key was present
- entrypoint contained the copy logic
- persistent debug file for the key existed
- `cipher-text.properties` still contained plain text values
- startup failed because secure-vault expected ciphertext-backed content

Conclusion:

- secret mounting worked
- ciphertext generation was missing
- this scenario is still blocked until the startup/image flow generates encrypted `cipher-text.properties`

Cleanup:

```bash
helm uninstall apim-sv-local -n apim-sv-good
kubectl delete ns apim-sv-good
```

## Scenario 4: Symmetric Secure Vault Negative With Pre-encrypted Values

This scenario depends on the same secure-vault startup path as scenario 3.

Create namespace and secret:

```bash
kubectl create ns apim-sv-negative
kubectl apply -n apim-sv-negative -f all-in-one/test-overrides/secrets/local-secure-vault-good.yaml
```

Install:

```bash
helm upgrade --install apim-sv-preenc ./all-in-one \
  -n apim-sv-negative \
  -f all-in-one/default_values.yaml \
  -f all-in-one/test-overrides/00-disable-gateway-api.yaml \
  -f all-in-one/test-overrides/00-image-prealpha-20260319-1.yaml \
  -f all-in-one/test-overrides/04-negative-secure-vault-preencrypted-inputs.yaml
```

Note:

- this test remains blocked by the same missing ciphertext generation/runtime handling found in scenario 3

Cleanup:

```bash
helm uninstall apim-sv-preenc -n apim-sv-negative
kubectl delete ns apim-sv-negative
```

## Scenario 5: Symmetric Secure Vault Negative With Wrong Master Key

Create namespace and wrong-master secret:

```bash
kubectl create ns apim-sv-wrong-master
kubectl apply -n apim-sv-wrong-master -f all-in-one/test-overrides/secrets/local-secure-vault-wrong-master.yaml
```

Install:

```bash
helm upgrade --install apim-sv-wrong ./all-in-one \
  -n apim-sv-wrong-master \
  -f all-in-one/default_values.yaml \
  -f all-in-one/test-overrides/00-disable-gateway-api.yaml \
  -f all-in-one/test-overrides/00-image-prealpha-20260319-1.yaml \
  -f all-in-one/test-overrides/05-negative-secure-vault-wrong-master-key.yaml
```

Note:

- this also depends on the same missing ciphertext generation/runtime handling as scenario 3

Cleanup:

```bash
helm uninstall apim-sv-wrong -n apim-sv-wrong-master
kubectl delete ns apim-sv-wrong-master
```

## Scenario 6: Asymmetric Secure Vault Backward Compatibility

Create namespace and local secret:

```bash
kubectl create ns apim-asym
kubectl apply -n apim-asym -f all-in-one/test-overrides/secrets/local-secure-vault-good.yaml
```

Install:

```bash
helm upgrade --install apim-asym ./all-in-one \
  -n apim-asym \
  -f all-in-one/default_values.yaml \
  -f all-in-one/test-overrides/00-disable-gateway-api.yaml \
  -f all-in-one/test-overrides/00-image-prealpha-20260319-1.yaml \
  -f all-in-one/test-overrides/06-asymmetric-backward-compatibility.yaml
```

Useful checks:

```bash
kubectl get pods -n apim-asym
kubectl get deploy -n apim-asym
kubectl get configmap -n apim-asym apim-asym-wso2am-all-in-one-conf-secret-conf -o jsonpath='{.data.secret-conf\.properties}'
kubectl logs -n apim-asym deploy/apim-asym-wso2am-all-in-one-am-deployment-1 --since=10m
```

Observed result:

- original version-based keystore path was wrong for `m2` packs
- chart was updated to avoid version-dependent lookup and to rewrite the secure-vault keystore path at runtime
- after that, startup moved past the old keystore-path error
- the next failure became decrypted password handling, again because `cipher-text.properties` still remained plain text

Conclusion:

- the keystore-path issue was a chart bug and was fixed
- the remaining blocker is still encrypted `cipher-text.properties` generation

Cleanup:

```bash
helm uninstall apim-asym -n apim-asym
kubectl delete ns apim-asym
```

## Practical Summary

What is working now:

- local secure-vault secret mounting
- runtime secret-conf keystore path correction for custom server home names
- symmetric key validation at Helm render time
- plain text symmetric non-secure-vault startup

What still needs work:

- generating encrypted `repository/conf/security/cipher-text.properties` before startup for both:
  - symmetric secure-vault mode
  - asymmetric secure-vault mode

Until that is implemented, secure-vault positive-path tests will continue to fail after mounting and config wiring succeed.
