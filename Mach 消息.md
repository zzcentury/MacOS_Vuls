# 简单消息

消息基本数据结构mach_msg_base_t

    typedef struct
    {
            mach_msg_header_t       header;
            mach_msg_body_t         body;
    } mach_msg_base_t;

消息头结构mach_msg_header_t

    typedef	struct 
    {
        mach_msg_bits_t	    msgh_bits;//消息头的标识位
        mach_msg_size_t	    msgh_size;//大小，以字节位单位
        mach_port_t		    msgh_remote_port;//目标端口
        mach_port_t		    msgh_local_port;//当前端口
        mach_port_name_t	msgh_voucher_port;//
        mach_msg_id_t		msgh_id;//唯一id
    } mach_msg_header_t;

消息体结构

    typedef struct
    {
            mach_msg_size_t msgh_descriptor_count;
    } mach_msg_body_t;

消息还可以选择带有一个消息尾

    typedef struct 
    {
        mach_msg_trailer_type_t	msgh_trailer_type;
        mach_msg_trailer_size_t	msgh_trailer_size;
    } mach_msg_trailer_t;

其中mach_msg_trailer_type_t类型

    typedef	unsigned int mach_msg_trailer_type_t;

# 复杂消息

将消息头的标志位mach_msg_bits_t设置为MACH_MSGH_BITS_COMPLEX，就表示复杂消息

数据结构与简单消息不同

在消息头后面接的是一个描述符计数字段，之后再接一个个描述符

    typedef struct
    {
        uint64_t			address;//指向数据的指针
        boolean_t     		deallocate: 8;//发送之后是否解除分配
        mach_msg_copy_options_t       copy: 8;//复制指令
        unsigned int     		pad1: 8;//
        mach_msg_descriptor_type_t    type: 8;//描述符
        mach_msg_size_t       	size;//在address处数据大小
    } mach_msg_ool_descriptor64_t;

其中mach_msg_descriptor_type_t已经定义的类型如下表

| trailer                         | 用途                               |
| :----------------------------:  | :----------------------------:    |
| MACH_MSG_PORT_DESCRIPTOR        | 传递一个端口权限                     |
| MACH_MSG_OOL_DESCRIPTOR         | 传递out-off-line数据                |
| MACH_MSG_OOL_PORTS_DESCRIPTOR   | 传递out-off-line端口                |
| MACH_MSG_OOL_VOLATILE_DESCRIPTOR| 传递有可能会发生变化的out-off-line数据 |

总的结构可以抽象为

    typedef struct
    {
            mach_msg_header_t       header;
            mach_msg_size_t         descriptor_num;//描述符计数
            mach_msg_ool_descriptor64_t descriptor[descriptor_num];//一个一个描述符
            mach_msg_body_t         body;
            mach_msg_trailer_t      trailer;
    } mach_msg_complex;

# 发送消息

消息的发送和接收都是同一个API

    extern mach_msg_return_t    
    mach_msg(
        mach_msg_header_t *msg,
        mach_msg_option_t option,//可以设置为收消息还是发消息等类型
        mach_msg_size_t send_size,
        mach_msg_size_t rcv_size,
        mach_port_name_t rcv_name,
        mach_msg_timeout_t timeout,
        mach_port_name_t notify
    );

# port

每一条Mach Message都是从一个port发送到另外一个port，而每一个port都有自己的权限。

消息从某个端口发送到另一个端口，每一个端口都可以接受来自任意端口的消息，但是只能有一个指定接受者

端口的权限如下

| MACH_PORT_RIGHT_| 含义                         |
| :------------:  | :----------------------:    |
| SEND            | 向这个端口发送消息，允许多个发送者|
| RECEIVE         | 从这个端口读取消息             |
| SEND_ONE        | 只能发送一次消息               |
| PORT_SET        | 同时拥有多个端口的接受权限      |
| DEAD_NAME       | 端口在SEND_ONE之后用完了权限   |

特殊的ports

    host port：代表正在运行该task的整台机器的port。
    task port: 正在运行的task本身的port。
    bootstrap port : 和bootstrap server连接着的一个port。

