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

# mig(Mach接口生成器)

# IPC

端口对象结构体

    struct ipc_port {

        /*
        * Initial sub-structure in common with ipc_pset
        * First element is an ipc_object second is a
        * message queue
        */
        struct ipc_object ip_object;
        struct ipc_mqueue ip_messages;//消息队列

        union {
            struct ipc_space *receiver;//指向接受者IPC空间的指针
            struct ipc_port *destination;//指向全局端口的指针
            ipc_port_timestamp_t timestamp;
        } data;

        union {
            ipc_kobject_t kobject;//这个端口背后的对象类型(在osfmk/kern/ipc_kobject.h文件中定义的IKOT_*常量)
            ipc_importance_task_t imp_task;
            ipc_port_t sync_qos_override_port;
        } kdata;
            
        struct ipc_port *ip_nsrequest;
        struct ipc_port *ip_pdrequest;
        struct ipc_port_request *ip_requests;
        union {
            struct ipc_kmsg *premsg;
            struct {
                sync_qos_count_t sync_qos[THREAD_QOS_LAST];
                sync_qos_count_t special_port_qos;
            } qos_counter;
        } kdata2;

        mach_vm_address_t ip_context;

        natural_t ip_sprequests:1,	/* send-possible requests outstanding */
            ip_spimportant:1,	/* ... at least one is importance donating */
            ip_impdonation:1,	/* port supports importance donation */
            ip_tempowner:1,	/* dont give donations to current receiver */
            ip_guarded:1,         /* port guarded (use context value as guard) */
            ip_strict_guard:1,	/* Strict guarding; Prevents user manipulation of context values directly */
            ip_specialreply:1,	/* port is a special reply port */
            ip_link_sync_qos:1,	/* link the special reply port to destination port */
            ip_impcount:24;	/* number of importance donations in nested queue */

        mach_port_mscount_t ip_mscount;
        mach_port_rights_t ip_srights;
        mach_port_rights_t ip_sorights;

    #if	MACH_ASSERT
    #define	IP_NSPARES		4
    #define	IP_CALLSTACK_MAX	16
    /*	queue_chain_t	ip_port_links;*//* all allocated ports */
        thread_t	ip_thread;	/* who made me?  thread context */
        unsigned long	ip_timetrack;	/* give an idea of "when" created */
        uintptr_t	ip_callstack[IP_CALLSTACK_MAX]; /* stack trace */
        unsigned long	ip_spares[IP_NSPARES]; /* for debugging */
    #endif	/* MACH_ASSERT */
    };

ipc_object

    struct ipc_object {
        ipc_object_bits_t io_bits;
        ipc_object_refs_t io_references;
        lck_spin_t	io_lock_data;
    };

ipc_mqueue

    typedef struct ipc_mqueue {
        union {
            struct {
                struct  waitq		waitq;
                struct ipc_kmsg_queue	messages;
                mach_port_seqno_t 	seqno;
                mach_port_name_t	receiver_name;
                uint16_t		msgcount;
                uint16_t		qlimit;
    #if MACH_FLIPC
                struct flipc_port	*fport;	// Null for local port, or ptr to flipc port
    #endif
            } port;
            struct {
                struct waitq_set	setq;
            } pset;
        } data;
        struct klist imq_klist;
    } *ipc_mqueue_t;

## 消息传递的实现

mach_msg函数

    /*
    *	Routine:	mach_msg
    *	Purpose:
    *		Send and/or receive a message.  If the message operation
    *		is interrupted, and the user did not request an indication
    *		of that fact, then restart the appropriate parts of the
    * 		operation.
    */
    mach_msg_return_t
    mach_msg(msg, option, send_size, rcv_size, rcv_name, timeout, notify)
        mach_msg_header_t *msg;
        mach_msg_option_t option;
        mach_msg_size_t send_size;
        mach_msg_size_t rcv_size;
        mach_port_t rcv_name;
        mach_msg_timeout_t timeout;
        mach_port_t notify;
    {
        mach_msg_return_t mr;

        /*
        * Consider the following cases:
        *	1) Errors in pseudo-receive (eg, MACH_SEND_INTERRUPTED
        *	plus special bits).
        *	2) Use of MACH_SEND_INTERRUPT/MACH_RCV_INTERRUPT options.
        *	3) RPC calls with interruptions in one/both halves.
        *
        * We refrain from passing the option bits that we implement
        * to the kernel.  This prevents their presence from inhibiting
        * the kernel's fast paths (when it checks the option value).
        */

        mr = MACH_MSG_TRAP(msg, option &~ LIBMACH_OPTIONS,
                send_size, rcv_size, rcv_name,
                timeout, notify);
        if (mr == MACH_MSG_SUCCESS)
            return MACH_MSG_SUCCESS;

        if ((option & MACH_SEND_INTERRUPT) == 0)
            while (mr == MACH_SEND_INTERRUPTED)
                mr = MACH_MSG_TRAP(msg,
                    option &~ LIBMACH_OPTIONS,
                    send_size, rcv_size, rcv_name,
                    timeout, notify);

        if ((option & MACH_RCV_INTERRUPT) == 0)
            while (mr == MACH_RCV_INTERRUPTED)
                mr = MACH_MSG_TRAP(msg,
                    option &~ (LIBMACH_OPTIONS|MACH_SEND_MSG),
                    0, rcv_size, rcv_name,
                    timeout, notify);

        return mr;
    }

    #define MACH_MSG_TRAP(msg, opt, ssize, rsize, rname, to, not) \
	 mach_msg_trap((msg), (opt), (ssize), (rsize), (rname), (to), (not))

    #define LIBMACH_OPTIONS	(MACH_SEND_INTERRUPT|MACH_RCV_INTERRUPT)

mach_msg_trap函数

    /*
    *	Routine:	mach_msg_trap [mach trap]
    *	Purpose:
    *		Possibly send a message; possibly receive a message.
    *	Conditions:
    *		Nothing locked.
    *	Returns:
    *		All of mach_msg_send and mach_msg_receive error codes.
    */

    mach_msg_return_t
    mach_msg_trap(
        struct mach_msg_overwrite_trap_args *args)
    {
        kern_return_t kr;
        args->rcv_msg = (mach_vm_address_t)0;

        kr = mach_msg_overwrite_trap(args);
        return kr;
    }

mach_msg_overwrite_trap_args结构体

    struct mach_msg_overwrite_trap_args {
        PAD_ARG_(user_addr_t, msg);
        PAD_ARG_(mach_msg_option_t, option);
        PAD_ARG_(mach_msg_size_t, send_size);
        PAD_ARG_(mach_msg_size_t, rcv_size);
        PAD_ARG_(mach_port_name_t, rcv_name);
        PAD_ARG_(mach_msg_timeout_t, timeout);
        PAD_ARG_(mach_msg_priority_t, override);
        PAD_ARG_8
        PAD_ARG_(user_addr_t, rcv_msg);  /* Unused on mach_msg_trap */
    };

mach_msg_overwrite_trap函数

    /*
    *	Routine:	mach_msg_overwrite_trap [mach trap]
    *	Purpose:
    *		Possibly send a message; possibly receive a message.
    *	Conditions:
    *		Nothing locked.
    *	Returns:
    *		All of mach_msg_send and mach_msg_receive error codes.
    */

    mach_msg_return_t
    mach_msg_overwrite_trap(
        struct mach_msg_overwrite_trap_args *args)
    {
        mach_vm_address_t	msg_addr = args->msg;
        mach_msg_option_t	option = args->option;
        mach_msg_size_t		send_size = args->send_size;
        mach_msg_size_t		rcv_size = args->rcv_size;
        mach_port_name_t	rcv_name = args->rcv_name;
        mach_msg_timeout_t	msg_timeout = args->timeout;
        mach_msg_priority_t override = args->override;
        mach_vm_address_t	rcv_msg_addr = args->rcv_msg;
        __unused mach_port_seqno_t temp_seqno = 0;

        mach_msg_return_t  mr = MACH_MSG_SUCCESS;
        vm_map_t map = current_map();//获得当前的VM空间(vm_map)

        /* Only accept options allowed by the user */
        option &= MACH_MSG_OPTION_USER;//#define MACH_MSG_OPTION_USER	 (MACH_SEND_USER | MACH_RCV_USER)

        if (option & MACH_SEND_MSG) {
            ipc_space_t space = current_space();//获得当前的IPC空间
            ipc_kmsg_t kmsg;

            KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_START);

            mr = ipc_kmsg_get(msg_addr, send_size, &kmsg);

            if (mr != MACH_MSG_SUCCESS) {
                KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_END, mr);
                return mr;
            }

            KERNEL_DEBUG_CONSTANT(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_LINK) | DBG_FUNC_NONE,
                        (uintptr_t)msg_addr,
                        VM_KERNEL_ADDRPERM((uintptr_t)kmsg),
                        0, 0,
                        0);

            mr = ipc_kmsg_copyin(kmsg, space, map, override, &option);//

            if (mr != MACH_MSG_SUCCESS) {
                ipc_kmsg_free(kmsg);
                KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_END, mr);
                return mr;
            }

            mr = ipc_kmsg_send(kmsg, option, msg_timeout);//发送消息

            if (mr != MACH_MSG_SUCCESS) {
                mr |= ipc_kmsg_copyout_pseudo(kmsg, space, map, MACH_MSG_BODY_NULL);
                (void) ipc_kmsg_put(kmsg, option, msg_addr, send_size, 0, NULL);
                KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_END, mr);
                return mr;
            }

        }

        if (option & MACH_RCV_MSG) {
            thread_t self = current_thread();
            ipc_space_t space = current_space();
            ipc_object_t object;
            ipc_mqueue_t mqueue;

            mr = ipc_mqueue_copyin(space, rcv_name, &mqueue, &object);
            if (mr != MACH_MSG_SUCCESS) {
                return mr;
            }
            /* hold ref for object */

            if ((option & MACH_RCV_SYNC_WAIT) && !(option & MACH_SEND_SYNC_OVERRIDE)) {
                ipc_port_t special_reply_port;
                __IGNORE_WCASTALIGN(special_reply_port = (ipc_port_t) object);
                /* link the special reply port to the destination */
                mr = mach_msg_rcv_link_special_reply_port(special_reply_port,
                        (mach_port_name_t)override);
                if (mr != MACH_MSG_SUCCESS) {
                    io_release(object);
                    return mr;
                }
            }

            if (rcv_msg_addr != (mach_vm_address_t)0)
                self->ith_msg_addr = rcv_msg_addr;
            else
                self->ith_msg_addr = msg_addr;
            self->ith_object = object;
            self->ith_rsize = rcv_size;
            self->ith_msize = 0;
            self->ith_option = option;
            self->ith_receiver_name = MACH_PORT_NULL;
            self->ith_continuation = thread_syscall_return;
            self->ith_knote = ITH_KNOTE_NULL;

            ipc_mqueue_receive(mqueue, option, rcv_size, msg_timeout, THREAD_ABORTSAFE);
            if ((option & MACH_RCV_TIMEOUT) && msg_timeout == 0)
                thread_poll_yield(self);
            return mach_msg_receive_results(NULL);
        }

        return MACH_MSG_SUCCESS;
    }

## 发送消息
1.调用current_space()获得当前的IPC空间
2.调用ipc_kmsg_get(msg_addr, send_size, &kmsg)分配一个message缓冲区,复制消息到缓冲区，初始化kmsg结构体

    /*
    *	Routine:	ipc_kmsg_get
    *	Purpose:
    *		Allocates a kernel message buffer.
    *		Copies a user message to the message buffer.
    *	Conditions:
    *		Nothing locked.
    *	Returns:
    *		MACH_MSG_SUCCESS	Acquired a message buffer.
    *		MACH_SEND_MSG_TOO_SMALL	Message smaller than a header.
    *		MACH_SEND_MSG_TOO_SMALL	Message size not long-word multiple.
    *		MACH_SEND_TOO_LARGE	Message too large to ever be sent.
    *		MACH_SEND_NO_BUFFER	Couldn't allocate a message buffer.
    *		MACH_SEND_INVALID_DATA	Couldn't copy message data.
    */

3.调用ipc_kmsg_copyin(kmsg, space, map, override, &option)复制消息的权限，将所有的out-of-line数据的内存复制到当前的map

    /*
    *	Routine:	ipc_kmsg_copyin
    *	Purpose:
    *		"Copy-in" port rights and out-of-line memory
    *		in the message.
    *
    *		In all failure cases, the message is left holding
    *		no rights or memory.  However, the message buffer
    *		is not deallocated.  If successful, the message
    *		contains a valid destination port.
    *	Conditions:
    *		Nothing locked.
    *	Returns:
    *		MACH_MSG_SUCCESS	Successful copyin.
    *		MACH_SEND_INVALID_HEADER
    *			Illegal value in the message header bits.
    *		MACH_SEND_INVALID_DEST	Can't copyin destination port.
    *		MACH_SEND_INVALID_REPLY	Can't copyin reply port.
    *		MACH_SEND_INVALID_MEMORY	Can't grab out-of-line memory.
    *		MACH_SEND_INVALID_RIGHT	Can't copyin port right in body.
    *		MACH_SEND_INVALID_TYPE	Bad type specification.
    *		MACH_SEND_MSG_TOO_SMALL	Body is too small for types/data.
    */

4.调用ipc_kmsg_send(kmsg, option, msg_timeout)发送消息

    /*
    *	Routine:	ipc_kmsg_send
    *	Purpose:
    *		Send a message.  The message holds a reference
    *		for the destination port in the msgh_remote_port field.
    *
    *		If unsuccessful, the caller still has possession of
    *		the message and must do something with it.  If successful,
    *		the message is queued, given to a receiver, destroyed,
    *		or handled directly by the kernel via mach_msg.
    *	Conditions:
    *		Nothing locked.
    *	Returns:
    *		MACH_MSG_SUCCESS	The message was accepted.
    *		MACH_SEND_TIMED_OUT	Caller still has message.
    *		MACH_SEND_INTERRUPTED	Caller still has message.
    *		MACH_SEND_INVALID_DEST	Caller still has message.
    */

## 接受消息

1.调用current_thread()获得当前的线程

2.调用current_space()获得当前的IPC空间

3.调用ipc_mqueue_copyin获得IPC队列

4.调用ipc_mqueue_receive从消息队列中取出消息

5.调用mach_msg_receive_results()

