# 在 TKE 中细粒度管理用户操作集群资源的权限

 

## 概述

本文将介绍 TKE 集群资源的权限控制， 通过相关使用步骤和实践示例来讲解在 TKE 中如何细粒度控制用户操作集群资源的权限。

### TKE 的授权模块

Kubernetes 官方支持 Node、RBAC、ABAC、Webhook 四种集群资源鉴权模块，可以使用多个授权模块，授权模块将按顺序检查，较早的模块具有更高的优先级来允许或拒绝请求，详情请参考官方文档 [Kubernetes 授权模块概述](https://kubernetes.io/zh/docs/reference/access-authn-authz/authorization/#授权模块) 。

在 TKE 中默认启用了 Node 和 RBAC 两种鉴权模式，开启参数 `--authorization-mode=Node,RBAC` 。

- **Node** - 节点鉴权是一种特殊用途的鉴权模式，专门对 kubelet 发出的 API 请求进行鉴权。了解有关使用节点授权模式的更多信息，请参阅[节点授权模式](https://kubernetes.io/zh/docs/reference/access-authn-authz/node/)。

- **RBAC** - 基于角色的访问控制（RBAC）是一种基于企业内个人用户的角色来管理对计算机或网络资源的访问的方法，是一套比強制权限控制和自由选定权限控制更为中性且具有灵活性的动态配置权限技术，也是 Kubernetes 主流的权限控制模型。要了解有关使用 RBAC 模式的更多信息，请参阅 [RBAC 授权模式](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/)。

TKE 工作节点上使用了 Node 授权模式来控制节点上的 kubelet 执行 API 操作， 但是 Node 授权模式比较有局限性，具体请参阅[节点授权模式](https://kubernetes.io/zh/docs/reference/access-authn-authz/node/)。当遇到实际用户授权控制场景时，就需要通过 RBAC 这种灵活的授权模式来完成。 



### 腾讯云访问管理

访问管理（Cloud Access Management，CAM）是腾讯云提供的一套 Web 服务，用于帮助客户安全地管理腾讯云账户的访问权限，资源管理和使用权限。 详情请参考产品文档 [腾讯云访问管理概述](https://cloud.tencent.com/document/product/598) 。



## TKE 集群资源权限控制概述

TKE 集群资源权限控制支持 WEB 控制台管理授权和自定义 YAML 权限绑定两种方式，

- TKE 控制台授权管理

  此方式可为多个不同的腾讯云子账号进行授权管理，配置简单方便，能满足大部分腾讯云用户权限控制场景。

- 自定义 YAML 权限绑定

  此方式一般在复杂和个性化的权限控制场景下使用，更具有灵活性。

下面将分别介绍下两种方式的特点、原理和使用方式。

### TKE 控制台授权管理

基于 [Kubernetes 用户认证](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/) 的方式，结合了腾讯云 CAM 和 Kubernetes RBAC 授权模式，为腾讯云子账户授权。详情请参考 [容器服务 概述 - 权限管理 - 文档中心 - 腾讯云](https://cloud.tencent.com/document/product/457/46104)。该授权模式下，可通过容器服务控制台及 kubectl 两种方式进行集群内资源访问，授权原理如下图所示：

![tian](https://main.qcloudimg.com/raw/3bd92966b4b7a5d6ac4e5e0463b0a20a.png)



接下来看看怎样在 TKE 控制台中对腾讯云子账号进行授权管理：

1. 在 TKE 控制台点击【 授权管理】菜单下任意子菜单的 【RBAC策略生成器】按钮，即可进入授权配置页面：

![image-20201021150618355](https://main.qcloudimg.com/raw/0d26625f20000e90279fd5507835fcb6.png)

子账户在CAM拥有 AcquireClusterAdminRole Action 权限之后可以通过此方式获取集群 Kubernetes RBAC 的 tke:admin 角色，即获得集群最高级别的管理员权限。

![image-20201021150404118](https://main.qcloudimg.com/raw/14041e5f0e8d17c828a0506b3d6bc3d2.png)

2. 从【子账户】列表中选择想要授权的腾讯云子账号（可多选） ，如下图所示：

![image-20201021151001674](https://main.qcloudimg.com/raw/b612d0017e698a9a697c4b4c675cfdb2.png)

也可以选择 【服务角色】选项，在需要给具有逻辑组的多个子账户授权时，可以复用 CAM 对用户的分组，比如某个用户组已经绑定了某个CAM角色，那么直接选择这个 CAM 角色进行授权，绑了这个角色的用户组成员都会给授权，不需要手动一个个勾选子账号，如下图：

![image-20201029193153093](https://main.qcloudimg.com/raw/5ae13cf60f1fd8e4f981b67a5502f804.png)



3. 选择完需要授权的账号后点击【下一步】，进入权限配置界面，选择给子账户授权的 Namespaces 列表，列表类型有两种：

- 选择【所有Namespaces】表示授予子账户所有命名空间的作用域权限，即集群权限。

- 选择特定的命名空间，比如 default 命名空间，表示授予子账户的 default 命名空间作用域的权限。

根据授权需求选择相应的 Namespaces 列表，如下图所示：

![image-20201029193322576](https://main.qcloudimg.com/raw/e212c2d9dba24399356870bbf9dcdd57.png)

4. 选择 Namespaces 列表类型后，需选择要授予的权限类型，TKE 默认提供的权限类型有【管理员】、【运维人员】、【开发人员】、【只读用户】和 【自定义权限】五种类型，权限类型列表如下图所示：

![image-20201029192808409](https://main.qcloudimg.com/raw/24c6e55c3113d32a883dcaefe1158478.png)

根据授权需求选择相应权限类型， 五种权限类型的说明如下：

- 管理员 对所有命名空间下资源的读写权限，拥有集群节点、存储卷、命名空间、配额的读写权限，可配置子账号和权限的读写权限

- 运维人员 对所有命名空间下资源的读写权限，拥有集群节点、存储卷、命名空间、配额的读写权限

- 开发人员 对所有命名空间或所选命名空间下控制台可见资源的读写权限

- 只读用户 对所有命名空间或所选命名空间下控制台可见资源的只读权限

- 自定义权限 自定义选择已有的 ClusterRole 设置权限

5. 若权限类型选择【自定义权限】，则需要选择当前已有的 ClusterRole 对象来完成自定义授权，参考文档 [自定义策略授权](https://cloud.tencent.com/document/product/457/46106)，如下图所示：

![image-20201029192939260](https://main.qcloudimg.com/raw/9296234dd7a4ea1544c2c8979f65025f.png)

6. 当命名空间和权限类型选择完成且检查无误后，点击【完成】按钮，会自动生成的 YAML 资源并触发授权，例如下图为子账户授予管理员权限时生成的 YAML清单：

![image-20201012164649732](https://main.qcloudimg.com/raw/58c57bbe27d03687763a2121c7c14c84.png)

至此完成了在 TKE 控制台为腾讯云子账户授权，可以登录被授权的子账号验证一下是否对集群资源有相应授权的访问权限。



### 自定义 YAML 权限绑定

使用原生的 [Kubernetes RBAC 授权模式](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/)，用户根据需求自定义 YAML 来管理授权。授权原理和说明如下：

- 权限对象（Role 或 ClusterRole）： Role 权限对象作用于特定命名空间， ClusterRole 权限对象可复用于多个命名空间 （Rolebinding）或授予整个集群权限（ClusterRoleBinding）。
- 授权对象（Subjects）： 权限授予的主体对象。
- 权限绑定（Rolebinding 或 ClusterRoleBinding）： 将权限对象和授权对象进行组合绑定，Rolebinding 作用于某个命名空间， ClusterRoleBinding 将作用于整个集群。



![RBAC](https://main.qcloudimg.com/raw/575562885683ceda2fb03d5ae0c59fd9.jpg)



如上图所示， Kubernetes RBAC 授权主要提供了三种权限绑定方式：

1. 作用于单个命名空间权限绑定

    RoleBinding 引用 Role 对象，只授予该命名空间下资源权限的绑定。

2. 多个命名空间复用集群权限绑定

   多个命名空间下不同的 Rolebinding 可引用同一个 ClusterRole 对象模版为各自命名空间授予相同模版权限。 

3. 拥有集群权限绑定

   ClusterRoleBinding 引用 ClusterRole 模版，为 Subjects 授予整个集群的权限。

除此之外，Kubernetes RBAC 从 1.9 开始，集群角色（ClusterRole）还可以通过使用 `aggregationRule` 组合其他 ClusterRoles 的方式来创建，在此不作赘述，有兴趣的话可参考官网文档 [Aggregated ClusterRoles]( https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles) 说明。



接下来将分别实践如何使用上述的三种权限绑定方式实现对用户的授权管理。

#### 作用于单个命名空间权限绑定

下面介绍如何在 TKE 中使用一个 ServiceAccount 类型用户实现作用于单个命名空间的权限绑定，使用下面 Shell 脚本内容将创建测试命名空间、测试  `ServiceAccount`   账户和设置集群访问凭证（token）认证 ： 

```bash
USERNAME='sa-acc' # 设置测试账户名
NAMESPACE='sa-test' # 设置测试命名空间名
CLUSTER_NAME='cluster_name_xxx' # 设置测试集群名
# 创建测试命名空间
kubectl  create namespace ${NAMESPACE} 
# 创建测试 ServiceAccount 账户
kubectl create sa ${USERNAME} -n ${NAMESPACE} 
# 获取 ServiceAccount 账户自动创建的 Secret token 资源名
SECRET_TOKEN=$(kubectl get sa ${USERNAME} -n ${NAMESPACE} -o jsonpath='{.secrets[0].name}')
# 获取 secrets 的明文 Token
SA_TOKEN=$(kubectl get secret ${SECRET_TOKEN} -o jsonpath={.data.token} -n sa-test | base64 -d)
# 使用获取到的明文 token 信息设置一个 token 类型的访问凭证
kubectl config set-credentials ${USERNAME} --token=${SA_TOKEN}
# 设置访问集群所需要的 context 条目
kubectl config set-context ${USERNAME} --cluster=${CLUSTER_NAME} --namespace=${NAMESPACE} --user=${USERNAME}
```

运行 `kubectl config get-contexts` 命令可以查看生成的 contexts 条目： 

![image-20201020105559159](https://main.qcloudimg.com/raw/40f7223c29d2c78b5e1671afe28933ba.png)

创建一个 `Role` 权限对象资源 sa-role.yaml：

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: sa-test # 指定 Namespace
  name: sa-role-test
rules: # 设置权限规则
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

再创建一个  `RoleBinding` 对象资源 sa-rb-test.yaml，如下面的权限绑定 YAML 表示：添加 ServiceAccount 类型的 sa-acc 用户在 sa-test 命名空间具有 sa-role-test（ Role 类型） 的权限。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sa-rb-test
  namespace: sa-test 
subjects:
- kind: ServiceAccount
  name: sa-acc
  namespace: sa-test # ServiceAccount 所在 Namespace 
  apiGroup: ""  # 默认 apiGroup 组为 rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: sa-role-test
  apiGroup: ""  # 默认 apiGroup 组为 rbac.authorization.k8s.io
```

从下图验证结果可以看出，当 Context 为 sa-context 时，默认命名空间是 sa-test， 且对 sa-test 命名空间下拥有 sa-role-test（Role）对象中配置的权限，但在 default 命名空间下不具有任何权限。

![image-20201020111456470](https://main.qcloudimg.com/raw/237a717756c26ed8ed654851c2d7aa01.png)



#### 多个命名空间复用集群权限绑定

下面介绍在TKE中如何使用多个命名空间复用集群权限绑定授权。

使用以下脚本创建使用 X509 自签证书认证的用户、证书签名请求（CSR）和证书审批允许信任，并设置集群资源访问凭证 Context。

```bash
USERNAME='role_user' # 设置需要创建的用户名
NAMESPACE='default' # 设置测试命名空间名
CLUSTER_NAME='cluster_name_xxx' # 设置测试集群名
# 使用 Openssl 生成自签证书 key
openssl genrsa -out ${USERNAME}.key 2048
# 使用 Openssl 生成自签证书CSR 文件, CN 代表用户名，O 代表组名
openssl req -new -key ${USERNAME}.key -out ${USERNAME}.csr -subj "/CN=${USERNAME}/O=${USERNAME}" 
# 创建 Kubernetes 证书签名请求（CSR）
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: ${USERNAME}
spec:
  request: $(cat ${USERNAME}.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - client auth
EOF
# 证书审批允许信任
kubectl certificate approve ${USERNAME}
# 获取自签证书 CRT
kubectl get csr ${USERNAME} -o jsonpath={.status.certificate} | base64 --decode > ${USERNAME}.crt
# 设置集群资源访问凭证（X509 证书）
kubectl config set-credentials ${USERNAME} --client-certificate=${USERNAME}.crt --client-key=${USERNAME}.key
# 设置 Context 集群、默认Namespace 等
kubectl config set-context  ${USERNAME} --cluster=${CLUSTER_NAME} --namespace=${NAMESPACE} --user=${USERNAME}
```

创建一个 ClusterRole 对象资源 test-clusterrole.yaml：

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: test-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list", "create"]
```

再创建一个  `RoleBinding` 对象资源 clusterrole-rb-test.yaml，如下面的权限绑定 YAML 表示：添加自签证书认证类型的 role_user 用户在 default 命名空间具有 test-clusterrole（ClusterRole 类型） 的权限。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: clusterrole-rb-test
  namespace: default 
subjects:
- kind: User
  name: role_user
  namespace: default # User 所在 Namespace 
  apiGroup: ""  # 默认 apiGroup 组为 rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: test-clusterrole
  apiGroup: ""  # 默认 apiGroup 组为 rbac.authorization.k8s.io
```

从下图验证结果可以看到在 Context 为 role_user 时， 默认命名空间为 default，且拥有 test-clusterrole 权限对象配置的规则的权限。

![image-20201020114653469](https://main.qcloudimg.com/raw/2d57f7719b3f8d1b187b41943e396bd5.png)

继续创建第二个  `RoleBinding` 对象资源 clusterrole-rb-test2.yaml，如下面的权限绑定 YAML 表示：添加自签证书认证类型的 role_user 用户在 default2 命名空间具有  test-clusterrole（ClusterRole 类型） 的权限。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: clusterrole-rb-test
  namespace: default2 
subjects:
- kind: User
  name: role_user
  namespace: default # User 所在 Namespace 
  apiGroup: ""  # 默认 apiGroup 组为 rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: test-clusterrole
  apiGroup: ""  # 默认 apiGroup 组为 rbac.authorization.k8s.io
```

从下图验证结果可以看出，在 default2 命名空间下， role_user 同样拥有 test-clusterrole 配置的权限规则，至此实现了多个命名空间复用集群权限的绑定。

![image-20201020114512915](https://main.qcloudimg.com/raw/b948a35d0e49f1f99084b2cf8e6b7eb9.png)



#### 拥有集群权限绑定

创建 `ClusterRoleBinding` 对象资源 clusterrole-crb-test3.yaml，如下面的权限绑定 YAML 表示：添加证书认证类型的 role_user 用户在整个集群具有 test-clusterrole（ClusterRole 类型） 的权限。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: clusterrole-crb-test
subjects:
- kind: User
  name: role_user
  namespace: default # User 所在 Namespace 
  apiGroup: ""  # 默认 apiGroup 组为 rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: test-clusterrole
  apiGroup: ""  # 默认 apiGroup 组为 rbac.authorization.k8s.io
```

从下图的测试结果可以看出，应用了权限绑定的 YAML 后，role_user 拥有了集群范围的 test-clusterrole 的权限。

![image-20201020141737129](https://main.qcloudimg.com/raw/5f3415f45bac5b622264fd4929e104a5.png)



## 总结

本文主要介绍了在 TKE 控制台对腾讯云子账号授权管理的使用方法和步骤，以及如何通过自定义 YAML 方式实践 Kubernetes RBAC 三种常用的权限绑定。

TKE 控制台授权管理结合了腾讯云访问权限管理和 Kubernetes RBAC 授权模式，界面配置简单方便，能满足大部分腾讯云子账号的权限控制场景，自定义 YAML 权限绑定方式一般在复杂和个性化的用户权限控制场景下使用，更具有灵活性，用户可根据实际授权需求选择合适的权限管理方式。



## 参考

- Kubernetes 授权模块概述：https://kubernetes.io/zh/docs/reference/access-authn-authz/authorization/
- RBAC 授权模式：https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/
- 节点授权模式：https://kubernetes.io/zh/docs/reference/access-authn-authz/node/
- 腾讯云访问管理概述：https://cloud.tencent.com/document/product/598
- Kubernetes 用户认证：https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/
- 容器服务 - 权限管理 - 文档中心 - 腾讯云：https://cloud.tencent.com/document/product/457/46104
- 自定义策略授权：(https://cloud.tencent.com/document/product/457/46106
- 组合 ClusterRoles ：  https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles
- 证书签名请求：https://kubernetes.io/zh/docs/reference/access-authn-authz/certificate-signing-requests/

