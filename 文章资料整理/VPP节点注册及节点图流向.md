# VPP节点注册及节点图流向

## **1.注册新结点**

### **1.1注册节点**

向VPP中注册新结点的方式比较简单，使用VPP中定义好的`VLIB_REGISTER_NODE`宏来声明新结点即可：

```text
VLIB_REGISTER_NODE(node_name, static)
```

### 1.2**VLIB_REGISTER_NODE 宏**

```text
#ifndef CLIB_MARCH_VARIANT
#define VLIB_REGISTER_NODE(x,...)                                       \
    __VA_ARGS__ vlib_node_registration_t x;                             \
    static void __vlib_add_node_registration_##x (void)                     \
    __attribute__((__constructor__)) ;                                  \
    static void __vlib_add_node_registration_##x (void)                     \
    {                                                                       \
        vlib_main_t * vm = vlib_get_main();                                 \
        x.next_registration = vm->node_main.node_registrations;             \
        vm->node_main.node_registrations = &x;                              \
    }                                                                       \
    static void __vlib_rm_node_registration_##x (void)                      \
    __attribute__((__destructor__)) ;                                   \
    static void __vlib_rm_node_registration_##x (void)                      \
    {                                                                       \
        vlib_main_t * vm = vlib_get_main();                                 \
        VLIB_REMOVE_FROM_LINKED_LIST (vm->node_main.node_registrations,     \
                                      &x, next_registration);               \
    }                                                                       \
    __VA_ARGS__ vlib_node_registration_t x
#else
#define VLIB_REGISTER_NODE(x,...)                                       \
    STATIC_ASSERT (sizeof(# __VA_ARGS__) != 7,"node " #x " must not be declared as static"); \
    static __clib_unused vlib_node_registration_t __clib_unused_##x
#endif
```

`VLIB_REGISTER_NODE`本质上是新建一个`vlib_node_registration_t`类型的结构体`node_name`，并且创建一个名为`__vlib_add_node_registration_node_name (void)`的构造函数。该构造函数采用头插法，将这个`node`挂在了`vm->node_main.node_registrations`链表上。
将宏展开后如下：

```text
{
    static vlib_node_registration_t node_name;

    //声明构造函数
    static void __vlib_add_node_registration_node_name (void)             __attribute__((__constructor__)); 

    //头插法加入到节点链表
    static void __vlib_add_node_registration_node_name (void)                     
    {                                                                       
        vlib_main_t * vm = vlib_get_main();                                 
        node_name.next_registration = vm->node_main.node_registrations;             
        vm->node_main.node_registrations = &node_name;                              
    }                 

    //声明析构函数
    static void __vlib_rm_node_registration_node_name (void)                       __attribute__((__destructor__));     

    //从节点链表上去除node_name节点
    static void __vlib_rm_node_registration_node_name (void)                      
    {                                                                       
        vlib_main_t * vm = vlib_get_main();                                 
        VLIB_REMOVE_FROM_LINKED_LIST (vm->node_main.node_registrations,     
                                      &node_name, next_registration);               
    }                                                                       
    static vlib_node_registration_t node_name;
}
```

### 1.3**vlib_node_registration_t 结构体**

`vlib_node_registration_t` 是保存节点信息的结构体：

```text
//注册node结点时使用，保存结点业务逻辑的函数地址，结点类型，结点状态，结点名称等
typedef struct _vlib_node_registration
{
    /* Vector processing function for this node. */
    //节点的功能函数，从下面注册的 node_fn_registrations 链表中选择一个优先级最高的最为该成员的值
    vlib_node_function_t *function;

    /* Node function candidate registration with priority */
    //节点功能函数链表，通过优先级节点函数候选注册
    vlib_node_fn_registration_t *node_fn_registrations;

    /* Node name. */
    //节点名字
    char *name;

    /* Name of sibling (if applicable). */
    //兄弟节点名字
    char *sibling_of;

    /* Node index filled in by registration. */
    //node 的 index，与 vlib_node_main_t->nodes 对应
    u32 index;

    /* Type of this node. */
    //节点类型
    vlib_node_type_t type;

    /* Error strings indexed by error code for this node. */
    //节点错误字符串存放的buffer
    char **error_strings;

    /* Buffer format/unformat for this node. */
    //这个节点的缓冲区格式
    format_function_t *format_buffer;
    unformat_function_t *unformat_buffer;

    /* Trace format/unformat for this node. */
    //这个节点的消息跟踪格式
    format_function_t *format_trace;
    unformat_function_t *unformat_trace;

    /* Function to validate incoming frames. */
    //用于验证传入frame
    u8 *(*validate_frame)(struct vlib_main_t *vm,
                          struct vlib_node_runtime_t *,
                          struct vlib_frame_t *f);

    /* Per-node runtime data. */
    //节点运行时数据，私有数据存储位置 
    void *runtime_data;

    /* Process stack size. */
    //进程堆栈大小
    u16 process_log2_n_stack_bytes;

    /* Number of bytes of per-node run time data. */
    //每个节点运行时数据的字节数
    u8 runtime_data_bytes;

    /* State for input nodes. */
    //输入节点的状态
    u8 state;

    /* Node flags. */
    //节点标志
    u16 flags;

    /* protocol at b->data[b->current_data] upon entry to the dispatch fn */
    //符合协议的报文 b->data[b->current_data] 进入调度fn 
    u8 protocol_hint;

    /* Size of scalar and vector arguments in bytes. */
    //标量和向量的大小(以字节为单位)
    //和frame中的scalar、vector的大小相对应。
    //scalar_size 一般都为0。除了 ethernet_input 节点指定为 sizeof (ethernet_input_frame_t)。
    //vector_size 是在 VLIB_REGISTER_NODE 注册节点时指定的，一般为sizeof(u32)。
    u16 scalar_size, vector_size;

    /* Number of error codes used by this node. */
    //此节点使用的错误代码数量
    u16 n_errors;

    /* Number of next node names that follow. */
    //该节点指向的下一个节点个数
    u16 n_next_nodes;

    /* Constructor link-list, don't ask... */
    //所有的next node通过该成员形成链表
    struct _vlib_node_registration *next_registration;

    /* Names of next nodes which this node feeds into. */
    //下一个节点数组，存储的是名字
    char *next_nodes[];
} vlib_node_registration_t;
```

### **1.4 节点函数**

```text
#define VLIB_NODE_FN(node)                      \
        uword CLIB_MARCH_SFX (node##_fn)();                 \
        static vlib_node_fn_registration_t                  \
        CLIB_MARCH_SFX(node##_fn_registration) =                \
                { .function = &CLIB_MARCH_SFX (node##_fn), };               \
        \
        static void __clib_constructor                      \
        CLIB_MARCH_SFX (node##_multiarch_register) (void)           \
        {                                   \
            //这里引用了一个node节点，其名字为宏的输入参数，
            //说明在定义节点和其处理函数的时候要求它们有一样的名字。
            extern vlib_node_registration_t node;                   \
            vlib_node_fn_registration_t *r;                 \
            r = & CLIB_MARCH_SFX (node##_fn_registration);          \
            //处理函数优先级，根据优先级选择最高优先级的处理函数
            r->priority = CLIB_MARCH_FN_PRIORITY();             \
            r->name = CLIB_MARCH_VARIANT_STR;                   \
            //将函数添加到其对应的节点链表中，从这里可以看出一个节点可以有多个处理函数，
            //在函数register_node中会选择一个优先级最高的函数作为节点的最终处理函数。
            r->next_registration = node.node_fn_registrations;          \
            node.node_fn_registrations = r;                 \
        }                                   \
        uword CLIB_CPU_OPTIMIZED CLIB_MARCH_SFX (node##_fn)
```

将宏展开后如下（以 `node_name` 示例）：

```text
{
    uword node_name_fn();                 

    //声明 vlib_node_fn_registration_t 结构体
    static vlib_node_fn_registration_t    node_name_fn_registration =             
    { 
        .function = &node_name_fn, 
    };                

     static void __clib_constructor node_name_multiarch_register(void)           
    {                                    
        //这里引用了一个node节点，其名字为宏的输入参数，
        //说明在定义节点和其处理函数的时候要求它们有一样的名字。
        extern vlib_node_registration_t node;                    
        vlib_node_fn_registration_t *r;     

        //处理函数优先级，根据优先级选择最高优先级的处理函数
        r = &node_name_fn_registration;        
        r->priority = clib_cpu_march_priority();             
        r->name = "node-name";        

        //将函数添加到其对应的节点链表中，从这里可以看出一个节点可以有多个处理函数，
        //在函数register_node中会选择一个优先级最高的函数作为节点的最终处理函数。
        r->next_registration = node.node_fn_registrations;            
        node.node_fn_registrations = r;                 
    }                                    
    uword node_name_fn();
}
```

### **1.5示例：**

```text
VLIB_REGISTER_NODE (ip4_input_node) = {
  .name = "ip4-input",
  .vector_size = sizeof (u32),
  .protocol_hint = VLIB_NODE_PROTO_HINT_IP4,

  .n_errors = IP4_N_ERROR,
  .error_strings = ip4_error_strings,

  .n_next_nodes = IP4_INPUT_N_NEXT,
  .next_nodes = {
    [IP4_INPUT_NEXT_DROP] = "error-drop",
    [IP4_INPUT_NEXT_PUNT] = "error-punt",
    [IP4_INPUT_NEXT_OPTIONS] = "ip4-options",
    [IP4_INPUT_NEXT_LOOKUP] = "ip4-lookup",
    [IP4_INPUT_NEXT_LOOKUP_MULTICAST] = "ip4-mfib-forward-lookup",
    [IP4_INPUT_NEXT_ICMP_ERROR] = "ip4-icmp-error",
    [IP4_INPUT_NEXT_REASSEMBLY] = "ip4-full-reassembly",
  },

  .format_buffer = format_ip4_header,
  .format_trace = format_ip4_input_trace,
};

VLIB_NODE_FN(ip4_input_node)(vlib_main_t *vm, vlib_node_runtime_t *node,
                             vlib_frame_t *frame)
{
    return ip4_input_inline(vm, node, frame, /* verify_checksum */ 1);
}
```

## **2.定义节点数据走向**

也就是当前的节点在 node graph 中的位置，其前向节点（Previous）或者后向节点（Next）的逻辑关系。
除了在节点注册中指定外，还可以使用以下几种方式：

### **2.1 使用feature机制**

`feature`机制本质上来说还是结点，只不过该结点可以在运行的时候通过命令进行配置是否打开或关闭，从而影响数据流的走向。

- arc类的注册

```text
VNET_FEATURE_ARC_INIT (ip4_unicast, static) =
{
  .arc_name = "ip4-unicast",
  .start_nodes = VNET_FEATURES ("ip4-input", "ip4-input-no-checksum"),
  .last_in_arc = "ip4-lookup",
  .arc_index_ptr = &ip4_main.lookup_main.ucast_feature_arc_index,
};
```

- 选择合适的arc类
  对新加入的结点进行管理，新的feature（即我们新建的结点）必须属于某个`arc`类，并作用于某个`interface`实体。
  通过`set interface feature ``arc [disable]`命令来开启或关闭该feature功能。

通常`arc`类的名字对应为其起点结点的名字，使用命令开启关闭`feature`功能能动态的改变数据的流向。
如果选择按照feature机制来加入结点的话需要注意以下几点：
VPP提供的arc类比较多，我们需要自己选择合适的arc来插入我们的结点。

- 在arc类上登记feature结点

```text
VNET_FEATURE_INIT (ip4_lookup, static) =
{
  .arc_name = "ip4-unicast",
  .node_name = "ip4-lookup",
  .runs_before = 0,    /* not before any other features */
};
```

arc_name：为我们选定的feature结点要插入的地方；
node_name：为我们自己新注册的结点名；
runs_before：说明新结点必须比某个feature结点先执行，通过新结点后的流可能流入下个feature结点也可能到达其他路径。

### **2. 1不使用feature机制**

不使用feature机制的话，结点间的关系相对来说更加静态，只能在编译的时候确定结点间的关系，不能在运行的时候进行改变，只能由系统提供的几个接口来插入节点；向这些入口登记函数后，后续的数据流将传到你定义的结点。
节点插入函数：

- 基于 vlib 的最底层函数接口

```text
//根据指定node的下一跳的node的index
uword vlib_node_add_next(vlib_main_t *vm, uword node, uword next_node);

/* Add next node to given node in given slot. */
//根据指定node的下一跳的node的index，将指定node的下一跳Node添加到指定node的指定slot中。
//其实主要是构建指定node及其兄弟node的next_nodes、prev_node_bitmap、next_slot_by_node。
uword vlib_node_add_next_with_slot(vlib_main_t *vm,
                                   uword node_index,
                                   uword next_node_index, uword slot);

//根据指定node的下一跳的node的name，将指定node的下一跳node添加到指定node的指定slot中。
//其实主要是构建指定node及其兄弟node的next_nodes、prev_node_bitmap、next_slot_by_node。
uword vlib_node_add_named_next_with_slot(vlib_main_t *vm,
        uword node, char *name, uword slot);

/* As above but adds to end of node's next vector. */
//根据指定node的下一跳的node的name，将指定node的下一跳node添加到指定node的slot的尾部。
//其实主要是构建指定node及其兄弟node的next_nodes、prev_node_bitmap、next_slot_by_node。
uword vlib_node_add_named_next(vlib_main_t *vm, uword node, char *name)
{
    return vlib_node_add_named_next_with_slot(vm, node, name, ~0);
}
```

- L1

```text
//将某个hw interface的rx数据重定向到某个结点，node_index为结点的index索引
vnet_hw_interface_rx_redirect_to_node (vnet_main_t *vnm, u32 hw_if_index, u32 node_index)
```

- L2、L3

```text
//在"ethernet-input"结点后插入特定type的结点
ethernet_register_input_type (vlib_main_t *vm, ethernet_type_t type, u32 node_index)
```

这里`type`包括`ethernet_type(0x806, ARP)、ethernet_type (0x8100, VLAN)、ethernet_type (0x800, IP4)`等二、三层协议。
具体支持的相关协议见`src/vnet/ethernet/types.def`文件。

- L4

```text
//在"ip4-local"结点后插入特定protocol的结点
ip4_register_protocol (u32 protocol, u32 node_index)
```

这里`protocol`包括`ip_protocol (6, TCP)、ip_protocol (17, UDP)`等四层协议。
具体支持的相关协议见`src/vnet/ip/protocols.def`文件。

- L5

```text
//在"ip4-udp-lookup"结点后插入特定dst_port的结点
udp_register_dst_port (vlib_main_t * vm, udp_dst_port_t dst_port, u32 node_index, u8 is_ip4)
```

这里`dst_port`包括`ip_port (WWW, 80)`等五层应用端口。
具体支持的相关端口见`src/vnet/ip/ports.def`文件。

原文链接：https://zhuanlan.zhihu.com/p/383352641 原文作者：sinocoder