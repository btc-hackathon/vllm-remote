# vllm-remote
ServingRuntime
```
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  annotations:
    opendatahub.io/accelerator-name: migrated-gpu
    opendatahub.io/apiProtocol: REST
    opendatahub.io/hardware-profile-name: migrated-gpu-mglzi-serving
    opendatahub.io/recommended-accelerators: '["nvidia.com/gpu"]'
    opendatahub.io/template-display-name: vLLM NVIDIA GPU ServingRuntime for KServe-Raghul-0.8.4
    opendatahub.io/template-name: vllm-cuda-runtime-0.8.4
    openshift.io/display-name: my-model
  name: my-model
  namespace: new-vllm
  labels:
    opendatahub.io/dashboard: 'true'
spec:
  annotations:
    prometheus.io/path: /metrics
    prometheus.io/port: '8080'
    serving.knative.dev/progress-deadline: 30m
  containers:
    - args:
        - '--port=8080'
        - '--model=/mnt/models'
        - '--served-model-name={{.Name}}'
      command:
        - python
        - '-m'
        - vllm.entrypoints.openai.api_server
      env:
        - name: HF_HOME
          value: /tmp/hf_home
      image: 'quay.io/modh/vllm@sha256:7e55a682d4e04ca882f9454615426aa3659806fdfa8627707f95ba926a9ac2f1'
      name: kserve-container
      ports:
        - containerPort: 8080
          protocol: TCP
      volumeMounts:
        - mountPath: /dev/shm
          name: shm
  multiModel: false
  supportedModelFormats:
    - autoSelect: true
      name: vLLM
  volumes:
    - emptyDir:
        medium: Memory
        sizeLimit: 2Gi
      name: shm
```


Inference Service
```
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  annotations:
    openshift.io/display-name: my-model
    serving.knative.openshift.io/enablePassthrough: 'true'
    serving.kserve.io/deploymentMode: Serverless
    sidecar.istio.io/inject: 'true'
    sidecar.istio.io/rewriteAppHTTPProbers: 'true'
  name: my-model
  namespace: new-vllm
  labels:
    opendatahub.io/dashboard: 'true'
spec:
  predictor:
    imagePullSecrets:
      - name: conn
    maxReplicas: 1
    minReplicas: 1
    model:
      args:
        - '--tensor-parallel-size=2'
        - '--max-model-len=10000'
        - '--trust-remote-code'
        - '--distributed-executor-backend=mp'
        - '--enable-auto-tool-choice'
        - '--tool-call-parser=llama3_json'
      modelFormat:
        name: vLLM
      name: ''
      resources:
        limits:
          cpu: '2'
          memory: 8Gi
          nvidia.com/gpu: '2'
        requests:
          cpu: '1'
          memory: 4Gi
          nvidia.com/gpu: '2'
      runtime: my-model
      storageUri: 'oci://registry.redhat.io/rhelai1/modelcar-llama-3-3-70b-instruct:1.5'
    tolerations:
      - effect: NoSchedule
        key: nvidia.com/gpu
        operator: Exists
```
