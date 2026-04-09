# Tekton learning

## Setting up Tekton
### Setting up Tekton pipeline
```
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

### Setting up Tekton triggers and inteceptors
```
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml
```

### Setting up Tekton chains
```
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/chains/latest/release.yaml
```

### Configure Tekton Chains to store the provenance metadata locally.
```
kubectl patch configmap chains-config -n tekton-chains \
-p='{"data":{"artifacts.oci.storage": "", "artifacts.taskrun.format":"in-toto", "artifacts.taskrun.storage": "tekton"}}'
```

## Retrieve and verify the artifact provenance
1. Get the PipelineRun UID:
```
export PR_UID=$(tkn pr describe --last -o  jsonpath='{.metadata.uid}')
```
2. Fetch the metadata and store it in a JSON file:
```
tkn pr describe --last \
-o jsonpath="{.metadata.annotations.chains\.tekton\.dev/signature-pipelinerun-$PR_UID}" \
| base64 -d > metadata.json
```
3. View the provenance:
```
cat metadata.json | jq -r '.payload' | base64 -d | jq .
```
4. To verify that the metadata hasn’t been tampered with, check the signature with cosign:
```
cosign verify-blob-attestation --insecure-ignore-tlog \
--key k8s://tekton-chains/signing-secrets --signature metadata.json \
--type slsaprovenance --check-claims=false /dev/null
```
