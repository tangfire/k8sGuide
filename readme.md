# 存储 - 给数据一个可靠的家

## 01. 存储分类

![123](./img/img_123.png)



## 02. configMap

配置信息的保存方式

### configMap - 模型

![124](./img/img_124.png)


### configMap - 定义

![125](./img/img_125.png)


### 多个服务之间的文件（如配置文件、静态资源、数据文件等）保持一致性的两种思想：共享和注入

![126](./img/img_126.png)


![127](./img/img_127.png)

- 多个不同的服务内部的文件达到一致
  - 共享
    - 每一次在读取文件的时候，都会发生网络IO
  - 注入  configmap
    - 一次注入后，多次读取不再消耗网络IO



---

在分布式系统或微服务架构中，确保多个服务之间的文件（如配置文件、静态资源、数据文件等）保持一致性是一个常见需求。以下是两种核心思想：**共享（Sharing）** 和 **注入（Injection）**，分别通过不同的机制实现文件一致性。

---

## 1. **共享（Sharing）思想**
**核心逻辑**：多个服务通过访问同一个共享存储或中间媒介来获取文件，确保所有服务读取的内容始终一致。  
**适用场景**：文件较大、更新频率低、需要多服务实时同步的场景（如配置文件、静态资源）。

### 实现方式：
#### （1）**共享存储卷（Volume）**
- **工具**：Kubernetes 的 `PersistentVolume (PV)`、`PersistentVolumeClaim (PVC)`，或云存储（如 AWS S3、NFS）。
- **原理**：将文件存储在共享卷中，多个服务挂载同一卷，直接读取同一份文件。
- **示例**：
  ```yaml
  # Kubernetes Deployment挂载共享PVC
  volumes:
    - name: shared-config
      persistentVolumeClaim:
        claimName: my-pvc  # 多个Pod共用同一个PVC
  ```

#### （2）**分布式文件系统**
- **工具**：HDFS、Ceph、GlusterFS。
- **原理**：文件在分布式系统中全局可见，服务通过统一接口访问。

#### （3）**中间件同步**
- **工具**：数据库（如 MySQL）、配置中心（如 Apollo、Nacos）。
- **原理**：将文件内容存入数据库或配置中心，服务定期拉取或监听变更。

### 优缺点：
| **优点**                     | **缺点**                     |
|------------------------------|------------------------------|
| 文件单点存储，强一致性       | 共享存储可能成为性能瓶颈     |
| 更新时只需修改一次           | 需要处理并发读写冲突         |
| 适合大文件或频繁读场景       | 依赖外部存储系统，复杂度高   |

---

## 2. **注入（Injection）思想**
**核心逻辑**：将文件内容通过某种机制主动分发到各个服务中，服务本地持有文件的副本。  
**适用场景**：文件较小、需要版本控制、或服务需离线运行的场景（如容器启动配置）。

### 实现方式：
#### （1）**配置即代码（Configuration as Code）**
- **工具**：Git 仓库 + CI/CD（如 Jenkins、GitLab CI）。
- **原理**：文件存储在代码库中，服务部署时通过流水线将文件打包到容器镜像或直接注入环境变量。
- **示例**：
  ```dockerfile
  # Dockerfile中复制配置文件
  COPY ./configs/app.conf /etc/service/app.conf
  ```

#### （2）**配置管理工具**
- **工具**：Ansible、Chef、Puppet。
- **原理**：工具将文件推送到各个服务的本地目录。

#### （3）**容器启动时注入**
- **工具**：Kubernetes 的 `ConfigMap`、`Secret`。
- **原理**：将文件内容定义为 `ConfigMap`，挂载为容器的卷或环境变量。
- **示例**：
  ```yaml
  # 将ConfigMap挂载为Pod的文件
  volumes:
    - name: config-volume
      configMap:
        name: app-config
  ```

### 优缺点：
| **优点**                     | **缺点**                     |
|------------------------------|------------------------------|
| 服务本地访问文件，性能高     | 更新需重新分发，可能延迟     |
| 无共享依赖，部署更简单       | 多副本间可能存在短暂不一致   |
| 适合敏感配置（如密钥）       | 文件版本管理需额外设计       |

---

## **对比与选型建议**
| **维度**         | **共享（Sharing）**          | **注入（Injection）**        |
|------------------|-----------------------------|-----------------------------|
| **一致性强度**   | 强一致性（实时同步）        | 最终一致性（需分发时间）     |
| **性能影响**     | 依赖共享存储性能            | 本地读取，性能高            |
| **复杂度**       | 需维护共享存储系统          | 需设计分发机制              |
| **适用场景**     | 多服务实时读写大文件        | 敏感配置或离线服务          |

### **实际组合使用案例**
- **Kubernetes 最佳实践**：
    - 用 `ConfigMap`（注入）管理配置文件，通过卷挂载到多个 Pod。
    - 用 `PV/PVC`（共享）存储日志或大数据文件。
- **混合架构**：
    - 数据库连接字符串通过 `Secret`（注入）分发，静态资源通过 CDN（共享）访问。

---

## **总结**
- **选择共享**：当需要强一致性、文件较大或多服务频繁读写时（如共享日志存储）。
- **选择注入**：当需要快速启动、隔离性高或配置敏感时（如微服务配置）。

两种思想并非互斥，实际系统中常结合使用，例如用共享存储存放大数据，同时用 ConfigMap 注入关键配置。



---


### **通俗版解释：共享 vs 注入**

想象一下，你和几个同事（代表不同的服务）需要一起完成一个项目，而项目资料（文件）需要保持一致。有两种方法可以做到：

---

### **1. 共享（Sharing）——大家一起看同一份资料**
**场景**：
- 你们所有人都在一个办公室里，资料放在办公室中央的一个共享文件夹里。
- 谁需要看资料，就直接去文件夹里拿。
- 如果有人更新了资料，其他人立刻就能看到最新版本。

**例子**：
- **云盘共享**：比如用百度网盘或Google Drive存文件，所有人访问同一个链接，文件一改大家都能看到。
- **Kubernetes共享存储**：多个服务挂载同一个磁盘（比如NFS），文件改了所有服务都能读到。

**优点**：
- 文件只有一份，不会乱（强一致性）。
- 改一次，所有人马上生效。

**缺点**：
- 如果共享文件夹挂了（比如网盘崩了），所有人都没法干活。
- 如果很多人同时改文件，可能会冲突（比如两个人同时改同一行）。

---

### **2. 注入（Injection）——每人发一份资料副本**
**场景**：
- 你们不在一个办公室，没法实时共享文件。
- 领导（系统管理员）把资料打印好，每人发一份，放各自桌上。
- 如果资料有更新，领导得重新打印，再发一遍（可能有点延迟）。

**例子**：
- **配置文件打包进软件**：比如APP的配置直接写在代码里，更新时要重新发布新版本。
- **Kubernetes ConfigMap**：把配置文件“注入”到每个服务里，改配置后要重启服务才能生效。

**优点**：
- 每人有自己的文件，不依赖别人（高可用）。
- 适合敏感信息（比如密码），因为不用共享存储。

**缺点**：
- 如果文件更新了，不能马上同步（得等领导重新发）。
- 如果每人手里的版本不一样，可能会乱（最终一致性）。

---

### **现实中的例子**
- **共享（Sharing）**
  - 公司用 **Excel 在线协作**（比如腾讯文档），所有人编辑同一份表格。
  - 风险：如果腾讯文档崩了，所有人都没法工作。

- **注入（Injection）**
  - 公司发 **PDF 文件**给每个员工，如果更新了得重新发邮件。
  - 风险：有人可能没收到最新版，还在用旧文件。

---

### **怎么选？**
| **方式** | **适合场景** | **不适用场景** |
|----------|------------|--------------|
| **共享** | 需要实时同步（如多人编辑文档） | 文件很大、共享存储不可靠 |
| **注入** | 需要独立运行（如APP配置） | 要求强一致性（不能容忍延迟） |

**实际应用**：
- **微服务配置**：用 **注入**（ConfigMap），因为每个服务要独立运行。
- **日志存储**：用 **共享**（NFS），因为日志要集中管理。

简单来说：
- **共享** = 大家一起看同一本书（实时，但依赖书不能丢）。
- **注入** = 每人发一本（独立，但更新慢）。

根据你的需求选就行！ 😊

---


### configMap - 创建

- `kubectl create configmap game-config --from-file=fire.file`
- `kubectl create configmap literal-config --from-literal=name=dave --from-literal=password=pass`

### **通俗解释：`kubectl create configmap game-config --from-file=fire.file`**

这条命令的作用是：**把本地的一个文件（`fire.file`）变成 Kubernetes 里的一个“配置包”（`ConfigMap`），名字叫 `game-config`，方便 Pod（容器）读取里面的内容。**

---

### **1. 什么是 `ConfigMap`？**
- **`ConfigMap` 是 Kubernetes 用来存配置的东西**，比如配置文件、环境变量等。
- 它不会存敏感信息（密码用 `Secret`），适合存普通配置，比如游戏参数、服务器地址等。
- Pod 可以像读文件或环境变量一样读取 `ConfigMap` 的内容。

---

### **2. `--from-file=fire.file` 是什么意思？**
- **`fire.file`**：是你电脑上的一个文件（比如内容可能是 `max_players=10`）。
- **`--from-file`**：告诉 Kubernetes “把这个文件的内容塞进 `ConfigMap` 里”。
- 文件内容会被存到 `ConfigMap` 的 `data` 字段下，键名默认是文件名（`fire.file`）。

---

### **3. 执行命令后会发生什么？**
- Kubernetes 会创建一个叫 `game-config` 的 `ConfigMap`。
- `fire.file` 的内容会被存进去，比如：
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: game-config
  data:
    fire.file: |  # 键名是文件名，值是文件内容
      max_players=10
      level=hard
  ```

---

### **4. Pod 怎么用这个 `ConfigMap`？**
#### **方式1：挂载成文件（最常用）**
让 Pod 把 `ConfigMap` 的内容变成一个文件（比如 `/etc/game/config`）：
```yaml
volumes:
  - name: config-volume
    configMap:
      name: game-config  # 使用刚创建的ConfigMap
containers:
  volumeMounts:
    - name: config-volume
      mountPath: /etc/game  # Pod内会看到文件：/etc/game/fire.file
```

#### **方式2：注入环境变量**
把 `ConfigMap` 的值变成环境变量：
```yaml
env:
  - name: MAX_PLAYERS  # 环境变量名
    valueFrom:
      configMapKeyRef:
        name: game-config
        key: fire.file  # 直接引用整个文件内容
```

---

### **5. 实际例子**
假设 `fire.file` 内容：
```ini
max_players=10
level=hard
```

**创建 ConfigMap：**
```bash
kubectl create configmap game-config --from-file=fire.file
```

**查看内容：**
```bash
kubectl get configmap game-config -o yaml
```
输出：
```yaml
data:
  fire.file: |
    max_players=10
    level=hard
```

---

### **6. 其他用法**
- **多个文件**：可以一次塞多个文件进去：
  ```bash
  kubectl create configmap my-config --from-file=file1 --from-file=file2
  ```
- **目录**：直接塞整个目录的文件：
  ```bash
  kubectl create configmap dir-config --from-file=./configs/
  ```

---

### **总结**
- **这条命令的作用**：把本地文件 `fire.file` 变成 Kubernetes 的 `ConfigMap`（`game-config`）。
- **Pod 怎么用**：挂载成文件或环境变量。
- **适用场景**：存游戏的配置、服务器参数等非敏感信息。

相当于把文件“上传”到 Kubernetes，让所有 Pod 都能方便地读取！ 🚀


---


### **通俗解释：`kubectl create configmap literal-config --from-literal=name=dave --from-literal=password=pass`**

这条命令的作用是：**直接在 Kubernetes 里创建一个叫 `literal-config` 的“配置包”（`ConfigMap`），并往里面存两对键值（`name=dave` 和 `password=pass`），方便 Pod（容器）读取这些值。**

---

### **1. 什么是 `ConfigMap`？**
- **`ConfigMap` 是 Kubernetes 用来存配置的东西**，比如环境变量、配置文件内容等。
- 它**不适合存敏感信息**（比如密码），敏感信息应该用 `Secret`。
- 这里只是举例，实际密码不要用 `ConfigMap` 存！

---

### **2. `--from-literal` 是什么意思？**
- **`--from-literal`**：直接告诉 Kubernetes “我要存一个键值对”。
- **`name=dave`**：键是 `name`，值是 `dave`。
- **`password=pass`**：键是 `password`，值是 `pass`（⚠️ 实际别这么干，用 `Secret`）。

---

### **3. 执行命令后会发生什么？**
Kubernetes 会创建一个 `ConfigMap`，内容如下：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: literal-config  # 你指定的名字
data:
  name: dave      # 键值对1
  password: pass  # 键值对2（⚠️ 实际别这么存密码！）
```

---

### **4. Pod 怎么用这个 `ConfigMap`？**
#### **方式1：作为环境变量（常用）**
让 Pod 读取 `ConfigMap` 的值变成环境变量：
```yaml
env:
  - name: PLAYER_NAME  # Pod内的环境变量名
    valueFrom:
      configMapKeyRef:
        name: literal-config  # ConfigMap名字
        key: name             # 读取的键（dave）
  - name: PLAYER_PASSWORD    # 另一个环境变量
    valueFrom:
      configMapKeyRef:
        name: literal-config
        key: password         # 读取的键（pass）
```

#### **方式2：挂载成文件**
把 `ConfigMap` 的键值对变成文件：
```yaml
volumes:
  - name: config-volume
    configMap:
      name: literal-config
containers:
  volumeMounts:
    - name: config-volume
      mountPath: /etc/config
```
Pod 内会看到：
```
/etc/config/name      # 内容：dave
/etc/config/password  # 内容：pass
```

---

### **5. 实际例子**
**创建 ConfigMap：**
```bash
kubectl create configmap literal-config --from-literal=name=dave --from-literal=password=pass
```

**查看内容：**
```bash
kubectl get configmap literal-config -o yaml
```
输出：
```yaml
data:
  name: dave
  password: pass
```

---

### **6. 注意事项**
1. **不要用 `ConfigMap` 存密码！**
  - 这个例子是为了演示，实际密码要用 `Secret`：
    ```bash
    kubectl create secret generic my-secret --from-literal=password=pass
    ```
2. **`--from-literal` vs `--from-file`**
  - `--from-literal`：直接写键值对（适合少量配置）。
  - `--from-file`：从文件加载（适合大配置）。

---

### **总结**
- **这条命令的作用**：直接创建一个 `ConfigMap`，并往里存 `name=dave` 和 `password=pass`。
- **Pod 怎么用**：可以变成环境变量或文件。
- **适用场景**：存普通配置（如用户名、服务器地址），**密码用 `Secret`**！

相当于往 Kubernetes 里“手写”了一个配置表，让 Pod 随便取用！ 🛠️


#### fire.file

```yaml
name=tangfire
password=123
```


```bash
[root@k8s-master01 ~]# vim fire.file
[root@k8s-master01 ~]# 
[root@k8s-master01 ~]# 
[root@k8s-master01 ~]# kubectl create configmap game-config --from-file=fire.file
configmap/game-config created
[root@k8s-master01 ~]# 
[root@k8s-master01 ~]# kubectl get cm
NAME               DATA   AGE
game-config        1      6s
kube-root-ca.crt   1      7d1h

```

```bash
[root@k8s-master01 ~]# kubectl get cm game-config -o yaml
apiVersion: v1
data:
  fire.file: |
    name=tangfire
    password=123
kind: ConfigMap
metadata:
  creationTimestamp: "2025-07-10T14:09:14Z"
  name: game-config
  namespace: default
  resourceVersion: "925246"
  uid: 4f99811f-996b-4294-89f6-b2149065b981

```


```bash
[root@k8s-master01 ~]# kubectl describe cm game-config
Name:         game-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
fire.file:
----
name=tangfire
password=123


BinaryData
====

Events:  <none>
```


```bash
[root@k8s-master01 ~]# kubectl create configmap literal-config --from-literal=name=dave --from-literal=password=pass
configmap/literal-config created
```

```bash
[root@k8s-master01 ~]# kubectl get cm
NAME               DATA   AGE
game-config        1      7m1s
kube-root-ca.crt   1      7d1h
literal-config     2      91s
[root@k8s-master01 ~]# kubectl get cm literal-config -o yaml
apiVersion: v1
data:
  name: dave
  password: pass
kind: ConfigMap
metadata:
  creationTimestamp: "2025-07-10T14:14:44Z"
  name: literal-config
  namespace: default
  resourceVersion: "925746"
  uid: 2defc3aa-84e3-4de5-9163-a076d2000436
```


### configMap - 环境变量

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: literal-config
  namespace: default
data:
  name: dave
  password: pass

```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
```


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-test-pod
spec:
  imagePullSecrets:
    - name: aliyun-regcred
  containers:
    - name: test-container
      image: crpi-cd1z0kbw072xy0ao.cn-guangzhou.personal.cr.aliyuncs.com/tangfire/myversion:v1
      command: ["/bin/sh", "-c", "env"]  # 启动后执行env命令显示所有环境变量
      env:
        - name: USERNAME  # 单个环境变量定义
          valueFrom:
            configMapKeyRef:
              name: literal-config  # 从名为literal-config的ConfigMap获取
              key: name             # 获取name键对应的值(dave)
        - name: PASSWORD  # 另一个环境变量
          valueFrom:
            configMapKeyRef:
              name: literal-config
              key: password         # 获取password键对应的值(pass)
      envFrom:
        - configMapRef:
            name: env-config  # 批量导入env-config ConfigMap的所有键值
  restartPolicy: Never  # 容器退出后不重启
```

#### ConfigMap - ENV(2.pod.yaml)



```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: literal-config
  namespace: default
data:
  name: dave
  password: pass

---

apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO


---

apiVersion: v1
kind: Pod
metadata:
  name: cm-env-test-pod
spec:
  imagePullSecrets:
    - name: aliyun-secret
  containers:
    - name: test-container
      image: crpi-cd1z0kbw072xy0ao.cn-guangzhou.personal.cr.aliyuncs.com/tangfire/myversion:v1
      command: ["/bin/sh", "-c", "env"]  # 启动后执行env命令显示所有环境变量
      env:
        - name: USERNAME  # 单个环境变量定义
          valueFrom:
            configMapKeyRef:
              name: literal-config  # 从名为literal-config的ConfigMap获取
              key: name             # 获取name键对应的值(dave)
        - name: PASSWORD  # 另一个环境变量
          valueFrom:
            configMapKeyRef:
              name: literal-config
              key: password         # 获取password键对应的值(pass)
      envFrom:
        - configMapRef:
            name: env-config  # 批量导入env-config ConfigMap的所有键值
  restartPolicy: Never  # 容器退出后不重启

```


```bash
[root@k8s-master01 7]# kubectl apply -f 2.pod.yaml 
configmap/literal-config created
configmap/env-config created
pod/cm-env-test-pod created
[root@k8s-master01 7]# kubectl get pod
NAME                    READY   STATUS      RESTARTS   AGE
cm-env-test-pod         0/1     Completed   0          5s
readiness-httpget-pod   0/1     Running     0          19h
[root@k8s-master01 7]# kubectl logs cm-env-test-pod
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.0.0.1:443
HOSTNAME=cm-env-test-pod
SHLVL=1
HOME=/root
MYAPP_SERVICE_HOST=10.13.161.153
PKG_RELEASE=1
MYAPP_PORT=tcp://10.13.161.153:80
MYAPP_SERVICE_PORT=80
USERNAME=dave
KUBERNETES_PORT_443_TCP_ADDR=10.0.0.1
NGINX_VERSION=1.25.5
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MYAPP_PORT_80_TCP_ADDR=10.13.161.153
KUBERNETES_PORT_443_TCP_PORT=443
NJS_VERSION=0.8.4
KUBERNETES_PORT_443_TCP_PROTO=tcp
MYAPP_SERVICE_PORT_80_80=80
MYAPP_PORT_80_TCP_PORT=80
NJS_RELEASE=3
MYAPP_PORT_80_TCP_PROTO=tcp
log_level=INFO
KUBERNETES_PORT_443_TCP=tcp://10.0.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.0.0.1
MYAPP_PORT_80_TCP=tcp://10.13.161.153:80
PWD=/
PASSWORD=pass
```


### configMap - 启动命令


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: literal-config
  namespace: default
data:
  name: dave
  password: pass

```

#### 3.pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-command-pod
spec:
  imagePullSecrets:
    - name: aliyun-secret
  containers:
    - name: myapp-container
      image: crpi-cd1z0kbw072xy0ao.cn-guangzhou.personal.cr.aliyuncs.com/tangfire/myversion:v1
      command: ["/bin/sh", "-c", "echo $(USERNAME) $(PASSWORD)"]
      env:
        - name: USERNAME
          valueFrom:
            configMapKeyRef:
              name: literal-config
              key: name
        - name: PASSWORD
          valueFrom:
            configMapKeyRef:
              name: literal-config
              key: password
  restartPolicy: Never
```

```bash
[root@k8s-master01 7]# kubectl apply -f 3.pod.yaml 
pod/cm-command-pod created
[root@k8s-master01 7]# kubectl get pod
NAME                    READY   STATUS      RESTARTS   AGE
cm-command-pod          0/1     Completed   0          5s
cm-env-test-pod         0/1     Completed   0          5m1s
readiness-httpget-pod   0/1     Running     0          19h
[root@k8s-master01 7]# kubectl logs cm-command-pod
dave pass
```


### configMap - 文件



```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-volume-pod
spec:
  imagePullSecrets:
    - name: aliyun-secret
  containers:
    - name: myapp-container
      image: crpi-cd1z0kbw072xy0ao.cn-guangzhou.personal.cr.aliyuncs.com/tangfire/myversion:v1
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: literal-config
  restartPolicy: Never
```

```bash
[root@k8s-master01 7]# kubectl apply -f 4.pod.yaml 
pod/cm-volume-pod created
[root@k8s-master01 7]# kubectl exec -it cm-volume-pod -- /bin/sh
/ # cd /etc/config
/etc/config # ls
name      password
/etc/config # cat name
dave/etc/config # cat password
pass/etc/config # ^C

/etc/config # exit
command terminated with exit code 130
[root@k8s-master01 7]# ^C
```


### configMap - 热更新 - 1

#### default.conf

```nginx
server {
    listen 80 default_server;
    server_name example.com www.example.com;
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
}
```

```bash
[root@k8s-master01 5]# kubectl create cm default-nginx --from-file=default.conf
configmap/default-nginx created
[root@k8s-master01 5]# kubectl get cm
NAME               DATA   AGE
default-nginx      1      18s

[root@k8s-master01 5]# kubectl create cm default-nginx --from-file=default.conf
configmap/default-nginx created
[root@k8s-master01 5]# kubectl get cm
NAME               DATA   AGE
default-nginx      1      18s
env-config         1      3h30m
game-config        1      15h
kube-root-ca.crt   1      7d16h
literal-config     2      3h30m
[root@k8s-master01 5]# ^C
[root@k8s-master01 5]# kubectl get cm default-nginx -o yaml
apiVersion: v1
data:
  default.conf: "server {\n    listen 80 default_server;\n    server_name example.com
    www.example.com;\n    \n    location / {\n        root /usr/share/nginx/html;\n
    \       index index.html index.htm;\n    }\n}\n"
kind: ConfigMap
metadata:
  creationTimestamp: "2025-07-11T05:34:46Z"
  name: default-nginx
  namespace: default
  resourceVersion: "1008811"
  uid: 7c1dfcda-8a7e-4fff-829f-c2b13f2f6c25
  
  
```




#### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hotupdate-deploy
  name: hotupdate-deploy
spec:
  replicas: 5
  selector:
    matchLabels:
      app: hotupdate-deploy
  template:
    metadata:
      labels:
        app: hotupdate-deploy
    spec:
      containers:
      - image: nginx:1.25
        name: nginx
        volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d/
      volumes:
      - name: config-volume
        configMap:
          name: default-nginx
```


```bash
[root@k8s-master01 5]# kubectl apply -f deployment.yaml 
deployment.apps/hotupdate-deploy created
[root@k8s-master01 5]# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
hotupdate-deploy-685f59b5d7-8pb9r   1/1     Running   0          6s
hotupdate-deploy-685f59b5d7-j62fw   1/1     Running   0          6s
hotupdate-deploy-685f59b5d7-mcphd   1/1     Running   0          6s
hotupdate-deploy-685f59b5d7-ql5wg   1/1     Running   0          6s
hotupdate-deploy-685f59b5d7-xvg9p   1/1     Running   0          6s
[root@k8s-master01 5]# ^C
[root@k8s-master01 5]# 
[root@k8s-master01 5]# 
[root@k8s-master01 5]# kubectl exec -it hotupdate-deploy-685f59b5d7-8pb9r -- /bin/bash
root@hotupdate-deploy-685f59b5d7-8pb9r:/# 
root@hotupdate-deploy-685f59b5d7-8pb9r:/# 
root@hotupdate-deploy-685f59b5d7-8pb9r:/# cd /etc/nginx/conf.d
root@hotupdate-deploy-685f59b5d7-8pb9r:/etc/nginx/conf.d# ls
default.conf
root@hotupdate-deploy-685f59b5d7-8pb9r:/etc/nginx/conf.d# cat default.conf 
server {
    listen 80 default_server;
    server_name example.com www.example.com;
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
}
```
再起一个终端，执行：

```bash
[root@k8s-master01 6]# kubectl edit cm default-nginx
configmap/default-nginx edited
```

将端口从80改成8080

```bash
[root@k8s-master01 6]# kubectl get cm default-nginx -o yaml
apiVersion: v1
data:
  default.conf: "server {\n    listen 8080 default_server;\n    server_name example.com
    www.example.com;\n    \n    location / {\n        root /usr/share/nginx/html;\n
    \       index index.html index.htm;\n    }\n}\n"
kind: ConfigMap
metadata:
  creationTimestamp: "2025-07-11T05:34:46Z"
  name: default-nginx
  namespace: default
  resourceVersion: "1011934"
  uid: 7c1dfcda-8a7e-4fff-829f-c2b13f2f6c25
```

```bash
root@hotupdate-deploy-685f59b5d7-8pb9r:/etc/nginx/conf.d# cat default.conf 
server {
    listen 8080 default_server;
    server_name example.com www.example.com;
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
}
```

```bash
[root@k8s-master01 5]# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE   READINESS GATES
hotupdate-deploy-685f59b5d7-8pb9r   1/1     Running   0          38m   10.244.58.202   k8s-node02   <none>           <none>
hotupdate-deploy-685f59b5d7-j62fw   1/1     Running   0          38m   10.244.85.245   k8s-node01   <none>           <none>
hotupdate-deploy-685f59b5d7-mcphd   1/1     Running   0          38m   10.244.58.204   k8s-node02   <none>           <none>
hotupdate-deploy-685f59b5d7-ql5wg   1/1     Running   0          38m   10.244.58.201   k8s-node02   <none>           <none>
hotupdate-deploy-685f59b5d7-xvg9p   1/1     Running   0          38m   10.244.85.244   k8s-node01   <none>           <none>
[root@k8s-master01 5]# curl 10.244.58.202
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@k8s-master01 5]# curl 10.244.58.202:8080
curl: (7) Failed to connect to 10.244.58.202 port 8080: Connection refused
```


### configMap - 热更新 - 2


![128](./img/img_128.png)


```bash
kubectl patch deployment hotupdate-deploy --patch '{"spec":{"template":{"metadata":{"annotations":{"version/config":"666666666"}}}}}'
```


```bash
[root@k8s-master01 5]# kubectl patch deployment hotupdate-deploy --patch '{"spec":{"template":{"metadata":{"annotations":{"version/config":"666666666"}}}}}'
deployment.apps/hotupdate-deploy patched

[root@k8s-master01 5]# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE    IP              NODE         NOMINATED NODE   READINESS GATES
hotupdate-deploy-7b4b64978c-2djbf   1/1     Running   0          115s   10.244.85.247   k8s-node01   <none>           <none>
hotupdate-deploy-7b4b64978c-7d79b   1/1     Running   0          116s   10.244.85.246   k8s-node01   <none>           <none>
hotupdate-deploy-7b4b64978c-m4jhm   1/1     Running   0          114s   10.244.58.209   k8s-node02   <none>           <none>
hotupdate-deploy-7b4b64978c-t8g8m   1/1     Running   0          116s   10.244.58.203   k8s-node02   <none>           <none>
hotupdate-deploy-7b4b64978c-w52dw   1/1     Running   0          116s   10.244.58.206   k8s-node02   <none>           <none>
[root@k8s-master01 5]# curl 10.244.85.247:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

[root@k8s-master01 5]# curl 10.244.85.247:80
curl: (7) Failed to connect to 10.244.85.247 port 80: Connection refused
```


### configMap - 不可改变



![129](./img/img_129.png)

```bash
kubectl edit cm default-nginx
```

添加`immutable: true`

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  default.conf: "server {\n    listen 8080 default_server;\n    server_name example.com
    www.example.com;\n    \n    location / {\n        root /usr/share/nginx/html;\n
    \       index index.html index.htm;\n    }\n}\n"
kind: ConfigMap
immutable: true
metadata:
  creationTimestamp: "2025-07-11T05:34:46Z"
  name: default-nginx
  namespace: default
  resourceVersion: "1011934"
  uid: 7c1dfcda-8a7e-4fff-829f-c2b13f2f6c25
```



- cm 如果修改为不可改变的状态，是不允许回退的，是不可逆的。
- 可以删除此cm，然后重新创建一个无不可改变标记的cm
- 优点：
  - 防止出现一些错误的修改
  - 减少对apiServer请求压力

  

 

## 03. Secret

编码而来的安全

### Secret - 定义

![130](./img/img_130.png)


### Secret - 特性

![131](./img/img_131.png)


### Secret - 类型

![132](./img/img_132.png)


### Secret - Opaque - 概念


![133](./img/img_133.png)

### Secret - Opaque - 创建



![134](./img/img_134.png)

```bash
[root@k8s-master01 5]# echo -n "tangfire" | base64
dGFuZ2ZpcmU=
[root@k8s-master01 5]# echo -n "dGFuZ2ZpcmU=" | base64 -d
tangfire[root@k8s-master01 5]# 
[root@k8s-master01 5]#  echo -n "123456" | base64
MTIzNDU2
```

#### 7.secret.yaml


```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: MTIzNDU2
  username: dGFuZ2ZpcmU=
```


```bash
[root@k8s-master01 5]# vim 7.secret.yaml 
[root@k8s-master01 5]# 
[root@k8s-master01 5]# kubectl apply -f 7.secret.yaml 
secret/mysecret created
[root@k8s-master01 5]# kubectl get secret
NAME            TYPE                             DATA   AGE
aliyun-secret   kubernetes.io/dockerconfigjson   1      2d22h
mysecret        Opaque                           2      5s
[root@k8s-master01 5]# kubectl describe secret mysecret
Name:         mysecret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  6 bytes
username:  8 bytes
```

```bash
[root@k8s-master01 5]# kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  password: MTIzNDU2
  username: dGFuZ2ZpcmU=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"MTIzNDU2","username":"dGFuZ2ZpcmU="},"kind":"Secret","metadata":{"annotations":{},"name":"mysecret","namespace":"default"},"type":"Opaque"}
  creationTimestamp: "2025-07-11T11:03:44Z"
  name: mysecret
  namespace: default
  resourceVersion: "1038681"
  uid: 79c4de3e-6b3a-48ce-9121-0b4110a8c910
type: Opaque
[root@k8s-master01 5]# echo -n "MTIzNDU2" | base64 -d
123456[root@k8s-master01 5]# 
```


### Secret - Opaque - ENV

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: opaque-secret-env
  name: opaque-secret-env-deploy
spec:
  replicas: 5
  selector:
    matchLabels:
      app: op-se-env-pod
  template:
    metadata:
      labels:
        app: op-se-env-pod
    spec:
      imagePullSecrets:
        - name: aliyun-secret
      containers:
      - name: myapp-container
        image: crpi-cd1z0kbw072xy0ao.cn-guangzhou.personal.cr.aliyuncs.com/tangfire/myversion:v1
        ports:
        - containerPort: 80
        env:
        - name: TEST_USER
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: username
        - name: TEST_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
```



```bash
[root@k8s-master01 5]# vim 8.deployment.yaml
[root@k8s-master01 5]# 
[root@k8s-master01 5]# 
[root@k8s-master01 5]# kubectl apply -f 8.deployment.yaml 
deployment.apps/opaque-secret-env-deploy created
```

```bash
[root@k8s-master01 5]# kubectl get pod
NAME                                        READY   STATUS    RESTARTS   AGE
opaque-secret-env-deploy-7b945c5f98-9mzzp   1/1     Running   0          43s
opaque-secret-env-deploy-7b945c5f98-f5252   1/1     Running   0          43s
opaque-secret-env-deploy-7b945c5f98-hxfcx   1/1     Running   0          43s
opaque-secret-env-deploy-7b945c5f98-qfwnj   1/1     Running   0          43s
opaque-secret-env-deploy-7b945c5f98-xrxw5   1/1     Running   0          43s
[root@k8s-master01 5]# kubectl exec -it opaque-secret-env-deploy-7b945c5f98-9mzzp -- /bin/sh 
/ # env
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.0.0.1:443
HOSTNAME=opaque-secret-env-deploy-7b945c5f98-9mzzp
SHLVL=1
HOME=/root
TEST_PASSWORD=123456
PKG_RELEASE=1
TEST_USER=tangfire
TERM=xterm
KUBERNETES_PORT_443_TCP_ADDR=10.0.0.1
NGINX_VERSION=1.25.5
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
NJS_VERSION=0.8.4
KUBERNETES_PORT_443_TCP_PROTO=tcp
NJS_RELEASE=3
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.0.0.1:443
KUBERNETES_SERVICE_HOST=10.0.0.1
PWD=/
/ # 
```


### Secret - Opaque - Volume


![135](./img/img_135.png)

#### 9.pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: secret-volume
  name: secret-volume-pod
spec:
  imagePullSecrets:
    - name: aliyun-secret
  volumes:
    - name: volumes12
      secret:
        secretName: mysecret
  containers:
    - image: crpi-cd1z0kbw072xy0ao.cn-guangzhou.personal.cr.aliyuncs.com/tangfire/myversion:v1
      name: myapp-container
      volumeMounts:
        - name: volumes12
          mountPath: /data
```

```bash
[root@k8s-master01 5]# vim 9.pod.yaml
[root@k8s-master01 5]# 
[root@k8s-master01 5]# 
[root@k8s-master01 5]# kubectl apply -f 9.pod.yaml 
pod/secret-volume-pod created
[root@k8s-master01 5]# 
[root@k8s-master01 5]# kubectl get pod
NAME                READY   STATUS    RESTARTS   AGE
secret-volume-pod   1/1     Running   0          27s
[root@k8s-master01 5]# kubectl exec -it secret-volume-pod -- /bin/sh
/ # 
/ # cd /data/
/data # ls
password  username
/data # cat username
tangfire/data # cat password
123456/data # 
```



### Secret - Opaque - Volume - 热更新


![136](./img/img_136.png)



### Secret - Opaque - Volume - 不可更改

![137](./img/img_137.png)




## 04. Downward API


容器在运行时从Kubernetes API服务器获取有关它们自身的信息


### Downward API - 存在的意义


![138](./img/img_138.png)


### Downward API - env案例

#### 12.pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-env-example
spec:
  containers:
    - name: my-container
      image: wangyanglinux/myapp:v1.0
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: my-container
              resource: requests.cpu
        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: my-container
              resource: limits.cpu
        - name: MEMORY_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: my-container
              resource: requests.memory
        - name: MEMORY_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: my-container
              resource: limits.memory
  restartPolicy: Never
```

```bash
[root@k8s-master01 5]# vim 12.pod.yaml
[root@k8s-master01 5]# 
[root@k8s-master01 5]# 
[root@k8s-master01 5]# kubectl apply -f 12.pod.yaml 
pod/downward-api-env-example created
[root@k8s-master01 5]# 
[root@k8s-master01 5]# 
[root@k8s-master01 5]# kubectl get pod
NAME                       READY   STATUS              RESTARTS   AGE
downward-api-env-example   0/1     ContainerCreating   0          6s
secret-volume-pod          1/1     Running             0          96m
[root@k8s-master01 5]# kubectl get pod
NAME                       READY   STATUS    RESTARTS   AGE
downward-api-env-example   1/1     Running   0          48s
secret-volume-pod          1/1     Running   0          96m
[root@k8s-master01 5]# kubectl exec -it downward-api-env-example -- /bin/bash
downward-api-env-example:/# 
downward-api-env-example:/# env
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_PORT=443
CHARSET=UTF-8
HOSTNAME=downward-api-env-example
CPU_REQUEST=0
POD_NAME=downward-api-env-example
POD_NAMESPACE=default
PWD=/
HOME=/root
LANG=C.UTF-8
KUBERNETES_PORT_443_TCP=tcp://10.0.0.1:443
TERM=xterm
SHLVL=1
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.0.0.1
POD_IP=10.244.58.213
CPU_LIMIT=4
MEMORY_LIMIT=3698946048
KUBERNETES_SERVICE_HOST=10.0.0.1
KUBERNETES_PORT=tcp://10.0.0.1:443
MEMORY_REQUEST=0
KUBERNETES_PORT_443_TCP_PORT=443
LC_COLLATE=C
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
_=/usr/bin/env
downward-api-env-example:/# 
```

### Downward API - volume案例

#### 13.pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-volume-example
spec:
  containers:
    - name: my-container
      image: wangyanglinux/myapp:v1.0
      resources:
        limits:
          cpu: "1"
          memory: "512Mi"
        requests:
          cpu: "0.5"
          memory: "256Mi"
      volumeMounts:
        - name: downward-api-volume
          mountPath: /etc/podinfo
  volumes:
    - name: downward-api-volume
      downwardAPI:
        items:
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "name"
            fieldRef:
              fieldPath: metadata.name
          - path: "namespace"
            fieldRef:
              fieldPath: metadata.namespace
          - path: "uid"
            fieldRef:
              fieldPath: metadata.uid
          - path: "cpuRequest"
            resourceFieldRef:
              containerName: my-container
              resource: requests.cpu
          - path: "memoryRequest"
            resourceFieldRef:
              containerName: my-container
              resource: requests.memory
          - path: "cpuLimit"
            resourceFieldRef:
              containerName: my-container
              resource: limits.cpu
          - path: "memoryLimit"
            resourceFieldRef:
              containerName: my-container
              resource: limits.memory
  restartPolicy: Never
```

### **通俗解释：Kubernetes Downward API 卷挂载配置**

这个 YAML 定义了一个 Pod，名为 `downward-api-volume-example`，它通过 **Downward API** 将 Pod 自身的元数据（如名称、命名空间、标签等）和资源限制（CPU、内存）以文件的形式挂载到容器内部（`/etc/podinfo`）。

---

## **1. 核心功能**
- **Downward API**：Kubernetes 提供的一种机制，允许 Pod **获取自身的运行时信息**（如 Pod 名称、IP、资源限制等）。
- **用途**：应用程序可以通过读取 `/etc/podinfo` 目录下的文件，动态获取 Pod 的元数据和资源配额，而无需硬编码或依赖外部查询。

---

## **2. 关键配置解析**
### **(1) Pod 基本信息**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-volume-example
```
- **`apiVersion: v1`**：使用 Kubernetes 核心 API。
- **`kind: Pod`**：定义的是一个 Pod 资源。
- **`metadata.name`**：Pod 名称是 `downward-api-volume-example`。

---

### **(2) 容器配置**
```yaml
spec:
  containers:
    - name: my-container
      image: wangyanglinux/myapp:v1.0
      resources:
        limits:
          cpu: "1"
          memory: "512Mi"
        requests:
          cpu: "0.5"
          memory: "256Mi"
```
- **`image: wangyanglinux/myapp:v1.0`**：容器使用的镜像。
- **`resources`**：定义容器的资源限制和请求：
  - **`limits`**（硬限制）：
    - CPU：最多 1 核
    - 内存：最多 512MB
  - **`requests`**（最低需求）：
    - CPU：至少 0.5 核
    - 内存：至少 256MB

---

### **(3) Downward API 卷挂载**
```yaml
volumeMounts:
  - name: downward-api-volume
    mountPath: /etc/podinfo  # 挂载到容器内的路径
```
- **`volumeMounts`**：将 `downward-api-volume` 挂载到容器的 `/etc/podinfo` 目录。
- **容器内会生成以下文件**：
  - `/etc/podinfo/annotations`（Pod 的注解）
  - `/etc/podinfo/labels`（Pod 的标签）
  - `/etc/podinfo/name`（Pod 名称）
  - `/etc/podinfo/namespace`（Pod 所在的命名空间）
  - `/etc/podinfo/uid`（Pod 的唯一 ID）
  - `/etc/podinfo/cpuRequest`（CPU 请求值）
  - `/etc/podinfo/memoryRequest`（内存请求值）
  - `/etc/podinfo/cpuLimit`（CPU 限制值）
  - `/etc/podinfo/memoryLimit`（内存限制值）

---

### **(4) Downward API 卷定义**
```yaml
volumes:
  - name: downward-api-volume
    downwardAPI:
      items:
        - path: "annotations"  # 文件名
          fieldRef:
            fieldPath: metadata.annotations  # 数据来源
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
        - path: "name"
          fieldRef:
            fieldPath: metadata.name
        - path: "namespace"
          fieldRef:
            fieldPath: metadata.namespace
        - path: "uid"
          fieldRef:
            fieldPath: metadata.uid
        - path: "cpuRequest"
          resourceFieldRef:
            containerName: my-container  # 指定容器
            resource: requests.cpu      # 获取 CPU 请求值
        - path: "memoryRequest"
          resourceFieldRef:
            containerName: my-container
            resource: requests.memory
        - path: "cpuLimit"
          resourceFieldRef:
            containerName: my-container
            resource: limits.cpu
        - path: "memoryLimit"
          resourceFieldRef:
            containerName: my-container
            resource: limits.memory
```
- **`downwardAPI`**：定义 Downward API 卷，将 Pod 信息写入文件。
- **`items`**：指定要暴露哪些信息，每个 `path` 对应一个文件名，`fieldRef` 或 `resourceFieldRef` 指定数据来源。

---

### **(5) 重启策略**
```yaml
restartPolicy: Never
```
- **`Never`**：Pod 退出后不会自动重启（适合一次性任务）。

---

## **3. 实际应用场景**
1. **日志收集**：应用程序可以读取 `/etc/podinfo/name` 获取 Pod 名称，用于日志标记。
2. **动态配置**：根据 Pod 的命名空间（`namespace`）或标签（`labels`）调整运行参数。
3. **资源监控**：通过 `cpuLimit`、`memoryLimit` 等文件，让应用知道自己的资源配额。

---

## **4. 示例：容器内查看文件**
```bash
kubectl exec downward-api-volume-example -- cat /etc/podinfo/name
# 输出：downward-api-volume-example

kubectl exec downward-api-volume-example -- cat /etc/podinfo/cpuLimit
# 输出：1
```

---

## **总结**
- **Downward API** 让 Pod 能获取自身信息，无需依赖外部查询。
- **挂载成文件** 后，应用程序可以直接读取，适用于动态配置、日志标记等场景。
- **资源限制**（CPU/内存）也能通过文件暴露，方便应用自适应调整。

这样设计的好处是 **解耦**，应用程序不需要硬编码 Pod 信息，而是动态获取，提高可移植性。 🚀

```bash
[root@k8s-master01 5]# vim 13.pod.yaml
[root@k8s-master01 5]# 
[root@k8s-master01 5]# 
[root@k8s-master01 5]# kubectl apply -f 13.pod.yaml 
pod/downward-api-volume-example created
[root@k8s-master01 5]# kubectl get pod
NAME                          READY   STATUS              RESTARTS   AGE
downward-api-env-example      1/1     Running             0          22m
downward-api-volume-example   0/1     ContainerCreating   0          3s
secret-volume-pod             1/1     Running             0          118m
[root@k8s-master01 5]# kubectl exec -it downward-api-volume-example /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
error: unable to upgrade connection: container not found ("my-container")
[root@k8s-master01 5]# kubectl exec -it downward-api-volume-example --  /bin/bash
downward-api-volume-example:/# cd /etc/podinfo/
downward-api-volume-example:/etc/podinfo# ls
annotations    cpuRequest     memoryLimit    name           uid
cpuLimit       labels         memoryRequest  namespace
downward-api-volume-example:/etc/podinfo# ls -l
total 0
lrwxrwxrwx    1 root     root            18 Jul 11 21:36 annotations -> ..data/annotations
lrwxrwxrwx    1 root     root            15 Jul 11 21:36 cpuLimit -> ..data/cpuLimit
lrwxrwxrwx    1 root     root            17 Jul 11 21:36 cpuRequest -> ..data/cpuRequest
lrwxrwxrwx    1 root     root            13 Jul 11 21:36 labels -> ..data/labels
lrwxrwxrwx    1 root     root            18 Jul 11 21:36 memoryLimit -> ..data/memoryLimit
lrwxrwxrwx    1 root     root            20 Jul 11 21:36 memoryRequest -> ..data/memoryRequest
lrwxrwxrwx    1 root     root            11 Jul 11 21:36 name -> ..data/name
lrwxrwxrwx    1 root     root            16 Jul 11 21:36 namespace -> ..data/namespace
lrwxrwxrwx    1 root     root            10 Jul 11 21:36 uid -> ..data/uid
downward-api-volume-example:/etc/podinfo# cat annotations 
cni.projectcalico.org/containerID="ab58059ec2c7afc061d7c50794cf459ef4ab2702a68f3611f3fe1eb613daa028"
cni.projectcalico.org/podIP="10.244.85.251/32"
cni.projectcalico.org/podIPs="10.244.85.251/32"
kubectl.kubernetes.io/last-applied-configuration="{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"name\":\"downward-api-volume-example\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"image\":\"wangyanglinux/myapp:v1.0\",\"name\":\"my-container\",\"resources\":{\"limits\":{\"cpu\":\"1\",\"memory\":\"512Mi\"},\"requests\":{\"cpu\":\"0.5\",\"memory\":\"256Mi\"}},\"volumeMounts\":[{\"mountPath\":\"/etc/podinfo\",\"name\":\"downward-api-volume\"}]}],\"restartPolicy\":\"Never\",\"volumes\":[{\"downwardAPI\":{\"items\":[{\"fieldRef\":{\"fieldPath\":\"metadata.annotations\"},\"path\":\"annotations\"},{\"fieldRef\":{\"fieldPath\":\"metadata.labels\"},\"path\":\"labels\"},{\"fieldRef\":{\"fieldPath\":\"metadata.name\"},\"path\":\"name\"},{\"fieldRef\":{\"fieldPath\":\"metadata.namespace\"},\"path\":\"namespace\"},{\"fieldRef\":{\"fieldPath\":\"metadata.uid\"},\"path\":\"uid\"},{\"path\":\"cpuRequest\",\"resourceFieldRef\":{\"containerName\":\"my-container\",\"resource\":\"requests.cpu\"}},{\"path\":\"memoryRequest\",\"resourceFieldRef\":{\"containerName\":\"my-container\",\"resource\":\"requests.memory\"}},{\"path\":\"cpuLimit\",\"resourceFieldRef\":{\"containerName\":\"my-container\",\"resource\":\"limits.cpu\"}},{\"path\":\"memoryLimit\",\"resourceFieldRef\":{\"containerName\":\"my-container\",\"resource\":\"limits.memory\"}}]},\"name\":\"downward-api-volume\"}]}}\n"
kubernetes.io/config.seen="2025-07-11T21:36:18.116842185+08:00"
kubernetes.io/config.source="api"downward-api-volume-example:/etc/podinfo# cat cpuRequest 
downward-api-volume-example:/etc/podinfo# 
downward-api-volume-example:/etc/podinfo# cat cpuRequest 
downward-api-volume-example:/etc/podinfo# cat memoryRequest 
268435456downward-api-volume-example:/etc/podinfo# cat namespace 
defaultdownward-api-volume-example:/etc/podinfo# 
```

```bash
defaultdownward-api-volume-example:/etc/podinfo# cat labels 
downward-api-volume-example:/etc/podinfo# 
```

```bash
[root@k8s-master01 6]# kubectl get pod --show-labels
NAME                          READY   STATUS    RESTARTS   AGE    LABELS
downward-api-env-example      1/1     Running   0          35m    <none>
downward-api-volume-example   1/1     Running   0          13m    <none>
secret-volume-pod             1/1     Running   0          131m   name=secret-volume
[root@k8s-master01 6]# kubectl label pod downward-api-volume-example domain=tangfire.com
pod/downward-api-volume-example labeled
```

```bash
downward-api-volume-example:/etc/podinfo# cat labels 
domain="tangfire.com"downward-api-volume-example:/etc/podinfo# 
```

所以，以卷的绑定方式可以热更新，只要你对当前pod发生的一些修改，都会被反馈到文件内部的变化


### Downward API - volume 相较于 env 优势

- 会保持热更新的特性
- 传递一个容器的资源字段到另一个容器中


### Downward API - 扩展

![139](./img/img_139.png)



以下是完整的 Kubernetes RBAC 和 Pod 定义代码：

### 1. RBAC 权限配置 (`1.RBAC.yaml`)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: test-api-cluster-admin-binding
subjects:
- kind: ServiceAccount
  name: test-api
  namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### 2. Pod 定义 (`2.pod.yaml`)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl
spec:
  serviceAccountName: test-api
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "9999"]
```

### 3. 容器内执行的 API 访问命令
```bash
# 获取访问凭证
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CAPATH="/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

# 调用 Kubernetes API
curl -H "Authorization: Bearer $TOKEN" --cacert $CAPATH \
https://kubernetes/api/v1/namespaces/$NS/pods
```

### 关键说明：
1. **RBAC 配置**：
  - 创建了 `ClusterRoleBinding`，将 `cluster-admin` 角色绑定到 `default` 命名空间下的 `test-api` 服务账号
  - 授予了最高权限（生产环境应遵循最小权限原则）

2. **Pod 配置**：
  - 使用 `tutum/curl` 镜像创建名为 `curl` 的 Pod
  - 指定使用 `test-api` 服务账号
  - 容器启动后执行 `sleep 9999` 保持运行

3. **API 访问流程**：
  - 自动挂载的服务账号令牌提供认证信息
  - 使用 CA 证书验证 API Server 身份
  - 通过 Bearer Token 认证访问当前命名空间的 Pod 列表

### 使用步骤：






## 05. volume

## 06. pv/pvc


## 07. storageClass


## 08. 插曲

