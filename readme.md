## 前置
### Step1. 安裝 `Microk8s`
  ```
  // 透過Snap安裝MicroK8s
  sudo snap install microk8s --classic --channel=1.28/stable
   
  // 檢查安裝狀態
  microk8s status --wait-ready
  ```
  - 在 MicroK8s 中，自帶的 kubectl 指令通常以 microk8s kubectl 的形式使用。
   ```
   //Snap 提供了別名功能，可以為 MicroK8s 的 kubectl 指令建立全域別名。
   sudo snap alias microk8s.kubectl kubectl

   //測試別名是否有效
   kubectl version

   //啟用microk8s的docker Registry
   microk8s enable registry

   //MicroK8s 預設使用 containerd 作為runtime時，需配置 containerd 支援 HTTP 的方式拉取 image
   //https://microk8s.io/docs/registry-private

   ```

### Step2. 安裝 `K9s`
  ```
  //從 GitHub 官方頁面下載最新版本的 K9s
  curl -LO https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz

  //解壓縮文件
  tar -xvzf k9s_Linux_amd64.tar.gz

  //移動執行檔
  sudo mv k9s /usr/local/bin/

  //賦予執行權限
  sudo chmod +x /usr/local/bin/k9s

  //驗證安裝
  k9s version
  ```
  - 如何將microk8s與k9s做好設定
  - MicroK8s 預設的 kubeconfig 檔案路徑與標準路徑 (~/.kube/config) 不同，可能導致 K9s 缺少物資配置 
  - K9s 需要透過 kubeconfig 檔案連接 Kubernetes 叢集。MicroK8s 的 kubeconfig 檔案通常位於 `/var/snap/microk8s/current/credentials/client.config。`，方式如下
  ```
  //產生kubeconfig檔到路徑預設
  microk8s config > ~/.kube/config

  //設定環境變數KUBECONFIG：確保KUBECONFIG指向正確的路徑
  export KUBECONFIG=~/.kube/config

  //設定永久保存到 .bashrc 或 .zshrc
  echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
  source ~/.bashrc

  //驗證kubeconfig配置：使用kubectl檢查叢集連線是否正常
  kubectl get nodes
  ```
### Step3. 安裝 `Kubernetes Operator`
Reloader 是一個 Kubernetes Operator，可監控 ConfigMap 和 Secret 的變更並自動刷新依賴這些資源的 Pod。
```
  // 新增 Helm 倉庫
  helm repo add stakater https://stakater.github.io/stakater-charts

  // 安裝reloader
  helm install reloader stakater/reloader
```

### Step4. 安裝 `Camel-k`
```
// 建立camel-k namespace
kubectl create namespace camel-k

// 採用helm的方式安裝
microk8s helm repo add camel-k https://apache.github.io/camel-k/charts/
microk8s helm install camel-k camel-k/camel-k -n camel-k

```
### Step5. 安裝 `Camel-k integrationPlatform`
```
apiVersion: camel.apache.org/v1
kind: IntegrationPlatform
metadata:
  labels:
    app: camel-k
  name: camel-k
  namespace: camel-k
spec:
  build:
    registry:
      address: 35.189.186.4:32000
      organization: camel-k
      insecure: true

```

## 結合ConfigMap佈署integration
- 相關yaml api可以參考 : https://camel.apache.org/camel-k/2.5.x/apis/camel-k.html:pod　

### Step1. 創建 ConfigMap
建立 demo-rest-configmap.yaml
  ```
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: demo-rest-configmap
      namespace: camel-k
    data:
      demo-rest.yaml: |
        - restConfiguration:
            apiContextPath: api-docs
            apiProperty:
              - key: "api.title"
                value: "會員資訊Api"
              - key: "api.version"
                value: 1.0.0
              - key: "api.description"
                value: 此Service用來進行會員資訊管理111.
              - key: "api.contact.name"
                value: tom
              - key: "api.contact.email"
                value: tom@example.com
            componentProperty:
              - key: exchangeCharset
                value: UTF-8
            component: platform-http
            contextPath: /api
            host: 0.0.0.0
            port: "8080"

        - rest:
            id: rest-77ef
            get:
              - id: get-b449999
                description: "此API用來查詢使用者資訊test"
                param:
                  - description: ID of the user
                    dataType: string
                    name: userid
                    type: path
                produces: application/json
                to: direct:direct-query-user-data-test
            path: /user/{userid}
        
        - route:
            id: route-364277
            from:
              id: from-104388
              uri: direct
              parameters:
                name: direct-query-user-data-test
              steps:
                - setBody:
                    simple: Hello Camel from ${header.userid}

  ```

### Step2. 創建 Integration
建立 demo-rest-Integration.yaml 並綁定 ConfigMap
  ```
    apiVersion: camel.apache.org/v1
    kind: Integration
    metadata:
      name: demo-rest-integration
      namespace: camel-k
      # 開啟configmap刷新偵測
      annotations:
        reloader.stakater.com/auto: "true"
    spec:
      traits:
        container:
          port: 8080 # container expose port
          servicePort: 8080 # service expose port
        service:
          configuration:
            nodePort: true
            type: NodePort
      sources:
        - name: demo-rest.yaml
          language: yaml
          contentRef: demo-rest-configmap #對應到configmap
          contentKey: demo-rest.yaml #對應到configmap內的key
      #加入依賴
      dependencies:
        - camel:platform-http    # 支援 REST API 的 HTTP 服務
        - camel:rest             # 支援 REST DSL
        - camel:core             # Camel 核心路由功能
        - camel:direct           # 支援 direct 路由端點
        - camel:openapi-java     # 支援 OpenAPI 文件生成
        - camel:jackson          # 支援 JSON 資料處理（可選）
  ```
### Step3. 佈署 ConfigMap 與 Integration
  ```
  kubectl apply -f demo-rest-configmap.yaml
  kubectl apply -f demo-rest-integration.yaml
  ```

## 移除integration
kubectl delete integration <integration-name> -n camel