# k8s-expose-pod-info-to-containers-by-envars-sample

## 前言

參考文件 https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/

今天要來實作 [Expose Pod Information to Containers Through Environment Variables](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information) 這個任務

k8s 可以透過兩種方式來把 Pod 與 Container 的設定值傳入執行中的 Container:

1. Environment variables
2. Volume Files

這兩個種方式統稱做 Downward API


## 佈署目標

1 透過 Environment variables 傳入 Pod 的設定值

2 透過 Environment variables 傳入 Container 的設定值

## 透過 Environment variables 傳入 Pod 的設定值

建立 dapi-envars-pod.yaml 如下：

```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-fieldref
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          printenv MY_NODE_NAME MY_POD_NAME MY_POD_NAMESPACE;
          printenv MY_POD_IP MY_POD_SERVICE_ACCOUNT;
          sleep 10;
        done;
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: MY_POD_SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
  restartPolicy: Never
```

建立一個 Pod

設定名稱為 dapi-envars-fieldref

設定 container image 使用 k8s.gcr.io/busybox

設定 container 執行命令為 sh -c

設定 container 執行參數如下
```shell=
while true; do
  echo -en '\n';
  printenv MY_NODE_NAME MY_POD_NAME MY_POD_NAMESPACE;
  printenv MY_POD_IP MY_POD_SERVICE_ACCOUNT;
  sleep 10;
done;
```
設定 container 環境變數 MY_NODE_NAME 為 Pod field spec.nodeName

設定 container 環境變數 MY_POD_NAME 為 Pod field metadata.name

設定 container 環境變數 MY_POD_NAMESPACE 為 Pod field metadata.namespace

設定 container 環境變數 MY_POD_IP 為 Pod field status.podIP

設定 container 環境變數 MY_POD_SERVICE_ACCOUNT 為 Pod field spec.serviceAccountName

建置 Pod 指令如下：

```shell=
kubectl apply -f dapi-envars-pod.yaml
```

查詢 Pod 狀態指令如下:

```shell=
kubectl get pods
```
![](https://i.imgur.com/2tgGbPg.png)

查看 Container log 指令如下

```shell=
kubectl logs dapi-envars-fieldref
```
![](https://i.imgur.com/gXUncoK.png)

透過互動式 terminal 進入 Container 指令如下

```shell=
kubectl exec -it dapi-envars-fieldref -- sh
```

執行指令
```shell=
printenv
```
![](https://i.imgur.com/l9o0OoV.png)


## 透過 Environment variables 傳入 Container 的設定值

建立 dapi-envars-container.yaml 如下：
```yaml=
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-resourcefieldref
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox:1.24
      command: [ "sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          printenv MY_CPU_REQUEST MY_CPU_LIMIT;
          printenv MY_MEM_REQUEST MY_MEM_LIMIT;
          sleep 10;
        done;
      resources:
        requests:
          memory: "32Mi"
          cpu: "125m"
        limits:
          memory: "64Mi"
          cpu: "250m"
      env:
        - name: MY_CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: requests.cpu
        - name: MY_CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: limits.cpu
        - name: MY_MEM_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: requests.memory
        - name: MY_MEM_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: limits.memory
  restartPolicy: Never
```

建立一個 Pod

設定名稱為 dapi-envars-resourcefieldref

設定 container image 使用 k8s.gcr.io/busybox:1.24

設定 container 執行命令為 sh -c

設定 container 執行參數如下
```shell=
while true; do
  echo -en '\n';
  printenv MY_CPU_REQUEST MY_CPU_LIMIT;
  printenv MY_MEM_REQUEST MY_MEM_LIMIT;
  sleep 10;
done;
```

設定 container resources 如下：
```yaml=
requests:
  memory: "32Mi"
  cpu: "125m"
limits:
  memory: "64Mi"
  cpu: "250m"
```

設定 container 環境變數 MY_CPU_REQUEST 為 Container resource field requests.cpu

設定 container 環境變數 MY_CPU_LIMIT 為 Container resource field limits.cpu

設定 container 環境變數 MY_MEM_REQUEST 為 Container resource field requests.memory

設定 container 環境變數 MY_MEM_LIMIT 為 Container resource field limits.memory

建置 Pod 指令如下：

```shell=
kubectl apply -f dapi-envars-container.yaml
```

查詢 Pod 狀態指令如下:

```shell=
kubectl get pods
```
![](https://i.imgur.com/kIUjyvH.png)

查看 Container log 指令如下

```shell=
kubectl logs  dapi-envars-resourcefieldref
```

![](https://i.imgur.com/1Z3x5ob.png)

## 後話

實務上對於需要在 container 內根據設定檔而做不同執行策略

Downward API 提供了這樣的彈性

後面將會使用這個 Downward API 來建制 IndexJob 來處理多個分配的 Task