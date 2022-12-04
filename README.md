# controller-runtime的pkg内的包


1. client：访问k8s集群的客户端
2. cache：
   1. 提供本地缓存的读客户端
   2. 提供响应更新事件的处理器
3. manager：
   1. 用于创建并启动controller
   2. 用于共享的依赖，例如clients，cache，schemes等给controller
4. controller：
   1. 通过响应事件（对象的创建，更新和删除）来实现一个kubernetes API并确保对象的状态符合定义的spec，这个过程称为reconcile。
   2. controller将被作为work queue用来处理reconcile.Request请求
   3. 控制器并不直接处理事件，而是将Request放入队列，并批量处理。因此
      1. controller需要提供一个reconciler来执行从work queue提取的工作
      2. controller需要配置watches？？？
5. webhook: admission webhook是扩展kubernetes API的机制，可以配置当对象的事件发生时，api-server会先发送admissionRequest，用于改变admissionReview请求中的对象并返回给api-server。
   1. mutation admission webhook：用于修改k8s原生或者CRD实例。
   2. validating admission webhook：用于校验k8s对象是否正确。
   3. admission webhook需要定义handlers来处理接收的admissionReview请求
6. Reconciler: 
   1. Reconciler是controller的方法，并且提供了对象的Name和Namespace
   2. Reconciler用于确保对象的state和spec相符合。例如为replicaset调用Reconciler，目标副本是5但只有3个，Reconciler会创建2个pod，并设置OwnerReference执行对象的replicaset和设置标签controller=true。因此
      1. Reconciler包含controller的业务逻辑
      2. Reconciler通常只为一个对象类型工作，例如只为replicaset工作，其他对象使用其他的controller。
      3. 如果需要从其他对象触发Reconciler，可以提供一个mapping映射，例如onwerreference，将触发协调的对象映射到正在协调的对象。
      4. Reconciler会提供对象的Name和Namespace来工作
      5. Reconciler不关心事件的内容和类型，它只始终关注对象的状态和期待是否一致
7. Source：resource.Source
   1. Source通过watch api为kubernetes提供事件流
   2. 开发者应该只使用已提供的Source实现，而不是总是用自己自定义的Source
8. EventHandler：handler.EventHandler
   1. 处理Request的一个或者多个对象的事件？？
   2. 可以将一个对象的事件映射到一个相同类型的对象的Request
   3. 可以将一个对象的事件映射到不同类型的，例如将pod事件映射到拥有replicaset的Request
   4. 可以将一个对象的事件映射到多个Reconciler。例如将节点事件映射到响应集群大小调整事件的对象
   5. 开发者应该只使用已提供的eventhandler，尽量不用自己自定义的eventhandler
      1. EnqueueRequestForObject
      2. EnqueueRequestForOwner
9. Predicate: 用于过滤事件
   1. 接收一个事件并返回boolean，如果true则入队
   2. 用户可以已存在的predicate或者自定义predicate
10. Source，EventHandler都是Controller.Watch的参数，Predicate是可选参数


controller补充
eventhandlers补充

# pod controller的过程:
1. source提供事件： &source.KindSource{&v1.Pod{}} -> (Pod foo/bar Create Event)
2. eventhandler将Request入队：&handler.EnqueueRequestForObject{} -> (reconcile.Request{types.NamespaceName{Name: "foo", Namespace: "bar"}})
3. Reconciler被调用并携带Request对象： Reconciler(reconcile.Request{types.NamespaceName{Name: "foo", Namespace: "bar"}})


下面介绍一个协调replicaset对象的controller, 它会响应pod和replicaset事件
1. 监听pod和replicaset的Source
   1. replicaset：handler.EnqueueRequestForObject - enqueue a Request with the ReplicaSet Namespace and Name.
   2. pod（由replicaset创建）： handler.EnqueueRequestForOwnerHandler - enqueue a Request with the Owning ReplicaSet Namespace and Name.
2. 协调replicaset来响应事件
   1. replicaset被创建：读取replicaset和pod，如果pod缺少则创建pod
   2. Reconciler被调用（pod创建事件），读取replicaset和pod，不做任何操作
   3. Reconciler被调用（pod被其他原因删除），读取replicaset和pod，并创建pod。


# watching和eventhandler：

controller可能会监听多个类型的对象，但它应该只协调一种类型的对象。

当一个类型的对象需要因为其他类型对象的改变而更新，EnqueueRequestsFromMapFunc可以用来将一种事件类型映射到另一个类型。
例如通过协调多种类型对象来响应集群大小调整的事件。


例如deployment controller可能会使用 EnqueueRequestForObject 和 EnqueueRequestForOwner：
1. 监听deployment事件，将deployment的name和namespace入队
2. 监听replicaset事件，将它关联的deployment的name和namespace入队

reconcile.Requests 在入队时会进行去重。同一个 ReplicaSet 的许多 Pod 事件可能只触发 1 个Reconciler调用，因为每个事件都会导致 Handler 尝试为 ReplicaSet 排队相同的reconcile.Request。
   
   
# controller编写提示

1. Reconciler的复杂度
   1. 尽量使用O(1)的复杂度来处理N个对象的请求，而不是使用O(n)的复杂度来一次性处理（例如一个对象关联其他多个对象）
   2. 例如需要更新所有service来响应新增node事件，应该是协调服务，而不是协调节点并更新服务。
2. 事件的多路复用
   1. 相同Name和namespace的reconcile.Requests在入队之前，会被批量和去重处理。这需要controller能够优雅的处理一个对象的大量事件。
   2. 例如replicaset的pod事件将被转换成replicaset的Name和Namespace，所以replicaset只会被Reconciler协调一次（即使有多个pod的多个事件）


# 总结

监听官方资源对象：
1. 创建manager
2. 通过manger创建controller 
3. controller需要监听（watch）对象并进行相关的协调（reconcile）。 
   1. 通过controller接口的watch方法监听一个或多个对象（可多次调用），watch的方法签名：`Watch(src source.Source, eventhandler handler.EventHandler, predicates ...predicate.Predicate) error`
   2. 根据业务逻辑创建协调对象实现Reconciler接口，并注册到manager。
4. 通过manager创建webhookServer
   1. 注册一个或多个handler来处理admission webhook（mutating或validating）。
5. 启动manager


自定义CRD对象：
1. 创建自定义对象的struct
   1. 定义TypeMeta，ObjectMeta，Status，Spec属性
   2. 实现Validator，Defaulter等接口
   3. 实现一些自定义的方法
2. 定义自定义对象的List struct
3. 声明自定义对象的Group和Version的scheme
4. （可选，推荐）通过代码生成器，自动生成自定义对象是deepcopy代码
5. 主流程
   1. 创建manager
   2. 将上面自定义对象的GV注册到manager的schema
   3. 根据业务逻辑创建协调对象实现Reconciler接口，并注册到manager。
   4. controller监听相关对象。
   5. 注册自定义对象的webhook方法
   6. 启动manager

# 参考：

1. https://github.com/kubernetes-sigs/controller-runtime/blob/master/examples/builtins/main.go
2. https://github.com/kubernetes-sigs/controller-runtime/tree/master/examples/crd
3. https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.13.1/pkg