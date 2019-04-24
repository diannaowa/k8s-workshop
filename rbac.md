##  RBAC--基于角色的访问控制
 
 
 基于角色的访问控制（Role-Based Access Control, 即”RBAC”）使用”rbac.authorization.k8s.io” API Group实现授权决策，允许管理员通过Kubernetes API动态配置策略。
 
 从Kubernetes 1.8开始已经处于`stable`,并且api version为`rbac.authorization.k8s.io/v1`	。
 
 要启用RBAC,需要在启动kube-apiserver的时候指定参数`--authorization-mode=RBAC`。
 
 
**API概述**
 
 本节将介绍RBAC API所定义的四种顶级类型。用户可以像使用其他Kubernetes API资源一样 （例如通过kubectl、API调用等）与这些资源进行交互。例如，命令kubectl create -f (resource).yml 可以被用于以下所有的例子，当然，读者在尝试前可能需要先阅读以下相关章节的内容。
 
 
 **Role与ClusterRole**
 
 在RBAC API中，一个角色包含了一套表示一组权限的规则。 权限以纯粹的累加形式累积（没有”否定”的规则）。 角色可以由命名空间（`namespace`）内的`Role`对象定义，而整个Kubernetes集群范围内有效的角色则通过`ClusterRole`对象实现。

一个`Role`对象只能用于授予对某一单一命名空间中资源的访问权限。 以下示例描述了”default”命名空间中的一个`Role`对象的定义，用于授予对pod的读访问权限：


	kind: Role
	apiVersion: rbac.authorization.k8s.io/v1
	metadata:
	  namespace: default
	  name: pod-reader
	rules:
	- apiGroups: [""] # "" indicates the core API group
	  resources: ["pods"]
	  verbs: ["get", "watch", "list"]
	  
	  
	  
	  
ClusterRole对象可以授予与Role对象相同的权限，但由于它们属于集群范围对象， 也可以使用它们授予对以下几种资源的访问权限：

* 集群范围资源（例如节点，即node）
* 非资源类型endpoint（例如”/healthz”）
* 跨所有命名空间的命名空间范围资源（例如pod，需要运行命令kubectl get pods --all-namespaces来查询集群中所有的pod）

下面示例中的`ClusterRole`定义可用于授予用户对某一特定命名空间，或者所有命名空间中的secret（取决于其绑定方式）的读访问权限：

	kind: ClusterRole
	apiVersion: rbac.authorization.k8s.io/v1
	metadata:
	  # "namespace" omitted since ClusterRoles are not namespaced
	  name: secret-reader
	rules:
	- apiGroups: [""]
	  resources: ["secrets"]
	  verbs: ["get", "watch", "list"]
	  
	  
**RoleBinding与ClusterRoleBinding**


角色绑定将一个角色中定义的各种权限授予一个或者一组用户。 角色绑定包含了一组相关主体（即subject)---包括用户(User)、用户组(Group)、或者服务账户(Service Account)以及对被授予角色的引用。 在命名空间中可以通过RoleBinding对象授予权限，而集群范围的权限授予则通过ClusterRoleBinding对象完成。



RoleBinding可以引用在同一命名空间内定义的Role对象。 下面示例中定义的RoleBinding对象在”default”命名空间中将”pod-reader”角色授予用户”jane”。 这一授权将允许用户”jane”从”default”命名空间中读取pod。


	# This role binding allows "jane" to read pods in the "default" namespace.
	kind: RoleBinding
	apiVersion: rbac.authorization.k8s.io/v1
	metadata:
	  name: read-pods
	  namespace: default
	subjects:
	- kind: User
	  name: jane # Name is case sensitive
	  apiGroup: rbac.authorization.k8s.io
	roleRef:
	  kind: Role
	  name: pod-reader
	  apiGroup: rbac.authorization.k8s.io
	  
	  
RoleBinding对象也可以引用一个ClusterRole对象用于在RoleBinding所在的命名空间内授予用户对所引用的ClusterRole中 定义的命名空间资源的访问权限。这一点允许管理员在整个集群范围内首先定义一组通用的角色，然后再在不同的命名空间中复用这些角色。

例如，尽管下面示例中的RoleBinding引用的是一个ClusterRole对象，但是用户”dave”（即角色绑定主体）还是只能读取”development” 命名空间中的secret（即RoleBinding所在的命名空间）。

	# This role binding allows "dave" to read secrets in the "development" namespace.
	kind: RoleBinding
	apiVersion: rbac.authorization.k8s.io/v1
	metadata:
	  name: read-secrets
	  namespace: development # This only grants permissions within the "development" namespace.
	subjects:
	- kind: User
	  name: dave # Name is case sensitive
	  apiGroup: rbac.authorization.k8s.io
	roleRef:
	  kind: ClusterRole
	  name: secret-reader
	  apiGroup: rbac.authorization.k8s.io
	  
	  
	  
最后，可以使用ClusterRoleBinding在集群级别和所有命名空间中授予权限。下面示例中所定义的ClusterRoleBinding 允许在用户组”manager”中的任何用户都可以读取集群中任何命名空间中的secret。


	# This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
	kind: ClusterRoleBinding
	apiVersion: rbac.authorization.k8s.io/v1
	metadata:
	  name: read-secrets-global
	subjects:
	- kind: Group
	  name: manager # Name is case sensitive
	  apiGroup: rbac.authorization.k8s.io
	roleRef:
	  kind: ClusterRole
	  name: secret-reader
	  apiGroup: rbac.authorization.k8s.io
	  
	  
	  
**对资源的引用**

大多数资源由代表其名字的字符串表示，例如”pods”，就像它们出现在相关API endpoint的URL中一样。然而，有一些Kubernetes API还 包含了”子资源”，比如pod的logs。在Kubernetes中，pod logs endpoint的URL格式为：

	GET /api/v1/namespaces/{namespace}/pods/{name}/log
	
	
	
在这种情况下，”pods”是命名空间资源，而”log”是pods的子资源。为了在RBAC角色中表示出这一点，我们需要使用斜线来划分资源 与子资源。如果需要角色绑定主体读取pods以及pod log，您需要定义以下角色：

	kind: Role
	apiVersion: rbac.authorization.k8s.io/v1
	metadata:
	  namespace: default
	  name: pod-and-pod-logs-reader
	rules:
	- apiGroups: [""]
	  resources: ["pods", "pods/log"]
	  verbs: ["get", "list"]
	  
通过resourceNames列表，角色可以针对不同种类的请求根据资源名引用资源实例。当指定了resourceNames列表时，不同动作 种类的请求的权限，如使用”get”、”delete”、”update”以及”patch”等动词的请求，将被限定到资源列表中所包含的资源实例上。 例如，如果需要限定一个角色绑定主体只能”get”或者”update”一个configmap时，您可以定义以下角色：

	kind: Role
	apiVersion: rbac.authorization.k8s.io/v1
	metadata:
	  namespace: default
	  name: configmap-updater
	rules:
	- apiGroups: [""]
	  resources: ["configmaps"]
	  resourceNames: ["my-configmap"]
	  verbs: ["update", "get"]
	  
	  
值得注意的是，如果设置了resourceNames，则请求所使用的动词不能是list、watch、create或者deletecollection。 由于资源名不会出现在create、list、watch和deletecollection等API请求的URL中，所以这些请求动词不会被设置了resourceNames 的规则所允许，因为规则中的resourceNames部分不会匹配这些请求。


**`ClusterRoles`集合**

从Kubernetes 1.9开始，可以使用`aggregationRule`关键字创建多个`ClusterRole`集合后的`ClusterRole`。该`ClusterRole`的rules是其匹配到的所有`ClusterRole`的rules的并集。如：

	kind: ClusterRole
	apiVersion: rbac.authorization.k8s.io/v1
	metadata:
	  name: monitoring
	aggregationRule:
	  clusterRoleSelectors:
	  - matchLabels:
	      rbac.example.com/aggregate-to-monitoring: "true"
	rules: [] # Rules are automatically filled in by the controller manager.
	
	
在这个例子中，`monitoring`中的rules将会被下面`ClusterRole`的rules所填充（其满足了上面定义的标签选择器`rbac.example.com/aggregate-to-monitoring: true`）：

	kind: ClusterRole
	apiVersion: rbac.authorization.k8s.io/v1
	metadata:
	  name: monitoring-endpoints
	  labels:
	    rbac.example.com/aggregate-to-monitoring: "true"
	# These rules will be added to the "monitoring" role.
	rules:
	- apiGroups: [""]
	  Resources: ["services", "endpoints", "pods"]
	  verbs: ["get", "list", "watch"]
	  
	  
	  
**一些角色定义的例子**


在以下示例中，我们仅截取展示了rules部分的定义。

允许读取core API Group中定义的资源”pods”：

	rules:
	- apiGroups: [""]
	  resources: ["pods"]
	  verbs: ["get", "list", "watch"]
	  
	  
	  
允许读写在”extensions”和”apps” API Group中定义的”deployments”：

	rules:
	- apiGroups: ["extensions", "apps"]
	  resources: ["deployments"]
	  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
	  
	  
	 
允许读取”pods”以及读写”jobs”：

	rules:
	- apiGroups: [""]
	  resources: ["pods"]
	  verbs: ["get", "list", "watch"]
	- apiGroups: ["batch", "extensions"]
	  resources: ["jobs"]
	  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
	  
	  
允许读取一个名为”my-config”的ConfigMap实例（需要将其通过RoleBinding绑定从而限制针对某一个命名空间中定义的一个ConfigMap实例的访问）：

	rules:
	- apiGroups: [""]
	  resources: ["configmaps"]
	  resourceNames: ["my-config"]
	  verbs: ["get"]
	  
	  
允许读取core API Group中的”nodes”资源（由于Node是集群级别资源，所以此ClusterRole定义需要与一个ClusterRoleBinding绑定才能有效）：

	rules:
	- apiGroups: [""]
	  resources: ["nodes"]
	  verbs: ["get", "list", "watch"]
	  
	  
允许对非资源endpoint “/healthz”及其所有子路径的”GET”和”POST”请求（此ClusterRole定义需要与一个ClusterRoleBinding绑定才能有效）：

	rules:
	- nonResourceURLs: ["/healthz", "/healthz/*"] # 在非资源URL中，'*'代表后缀通配符
	  verbs: ["get", "post"]
	  
	  
	  
**对角色绑定主体（Subject）的引用**


`RoleBinding`或者`ClusterRoleBinding`将角色绑定到角色绑定主体（Subject）。 角色绑定主体可以是用户组（Group）、用户（User）或者服务账户（Service Accounts）。

用户由字符串表示。可以是纯粹的用户名，例如”alice”、电子邮件风格的名字，如 “bob@example.com” 或者是用字符串表示的数字id。由Kubernetes管理员配置认证模块 以产生所需格式的用户名。对于用户名，RBAC授权系统不要求任何特定的格式。然而，前缀`system:`是 为Kubernetes系统使用而保留的，所以管理员应该确保用户名不会意外地包含这个前缀。

Kubernetes中的用户组信息由授权模块提供。用户组与用户一样由字符串表示。Kubernetes对用户组 字符串没有格式要求，但前缀`system:`同样是被系统保留的。

服务账户拥有包含 `system:serviceaccount:`前缀的用户名，并属于拥有`system:serviceaccounts:`前缀的用户组。


**角色绑定的一些例子**

以下示例中，仅截取展示了RoleBinding的subjects字段。

一个名为”alice@example.com”的用户：

	subjects:
	- kind: User
	  name: "alice@example.com"
	  apiGroup: rbac.authorization.k8s.io
	  
	  
一个名为”frontend-admins”的用户组：

	subjects:
	- kind: Group
	  name: "frontend-admins"
	  apiGroup: rbac.authorization.k8s.io


kube-system命名空间中的默认服务账户：

	subjects:
	- kind: ServiceAccount
	  name: default
	  namespace: kube-system
	  
	  
`qa` namespace中所有的serviceaccount:

	subjects:
	- kind: Group
	  name: system:serviceaccounts:qa
	  apiGroup: rbac.authorization.k8s.io
	  
	  
集群中所有的serviceaccount:

	subjects:
	- kind: Group
	  name: system:serviceaccounts
	  apiGroup: rbac.authorization.k8s.io
	  
	  
所有认证过的用户：

	subjects:
	- kind: Group
	  name: system:authenticated
	  apiGroup: rbac.authorization.k8s.io
	  
	  
所有未认证的用户：

	subjects:
	- kind: Group
	  name: system:unauthenticated
	  apiGroup: rbac.authorization.k8s.io
	  
	  
所有用户：

	subjects:
	- kind: Group
	  name: system:authenticated
	  apiGroup: rbac.authorization.k8s.io
	- kind: Group
	  name: system:unauthenticated
	  apiGroup: rbac.authorization.k8s.io
