# ganesha owner

ganesha中不仅通过clientid来区分不同的client mount，一个文件系统同样要区分出对应的owner，即进程信息，特别是NFS4之后成为了一个有状态的协议。

## owner

ganesha由纯C编写，这更像是一个父类，下面细分openowner和lockowner以及clientowner。

## open owner

在ganesha上的结构体为：

```cpp
struct state_owner_t {
	state_owner_type_t so_type;
	struct glist_head so_lock_list;	
	pthread_mutex_t so_mutex;
	int32_t so_refcount;	
	int so_owner_len;	
	char *so_owner_val;	
	union {
		state_nfs4_owner_t so_nfs4_owner;
	} so_owner;
};

struct state_nfs4_owner_t {
	clientid4 so_clientid;
	nfs_client_id_t *so_clientrec;	
	bool so_confirmed;
	seqid4 so_seqid;			// 没找到自增位置？
	nfs_argop4_state so_args;
	void *so_last_entry;	
	nfs_resop4 so_resp;
	state_owner_t *so_related_owner;
	struct glist_head so_state_list; 
	struct glist_head so_perclient; 
	struct glist_head so_cache_entry;
	time_t so_cache_expire;
};
```

这里我们只讨论NFS协议，所以将比较无关的代码去掉，里面有一些我们比较感兴趣的字段：

#### state_owner_type_t   so_type

这是一个关于state类型的，有3种，即：

* STATE_OPEN_OWNER_NFSV4
* STATE_LOCK_OWNER_NFSV4
* STATE_CLIENTID_OWNER_NFSV4

#### so_owner_val/so_owner_len

这是一个标识符，客户端通过在请求种附带这个标识符，ganesha维护一个标识符到结构体指针的映射方案以提供客户端查询。

#### seqid4 so_seqid

用于保证幂等性， __在初始化过程中，seqid为0，随后的状态改变操作均需要携带seqid以便进行幂等性判断__。

关于openowner的创建，ganesha提供了一个接口：

```cpp
bool open4_open_owner(struct nfs_argop4 *op, compound_data_t *data,
		      struct nfs_resop4 *res, nfs_client_id_t *clientid,
		      state_owner_t **owner);
```

通过客户端提供的clientid、owner_val/owner_len到全局哈希表中寻找，如果没有，那么就会创建出来，seqid = 0，如果找到了，那么ganesha会验证其合法性，即`args->seqid = owner->seqid + 1`，但并不会改变这个seqid。

### state

讲完了owner，自然就是state了，owner拥有这些state，在ganesha中的state长这样：

```cpp
struct state_t {
	struct glist_head state_list;
	struct glist_head state_owner_list; 
	struct glist_head state_export_list;
	pthread_mutex_t state_mutex; 
	struct gsh_export *state_export;
	state_owner_t *state_owner;	 		// *
	struct fsal_obj_handle *state_obj;  // *
	struct fsal_export *state_exp; 
	union state_data state_data;
	enum state_type state_type;
	u_int32_t state_seqid;		// *
	int32_t state_refcount;		
	char stateid_other[OTHERSIZE];	
	struct state_refer state_refer;	
};

union state_data {
	struct state_share share; // share number for open
	struct state_nlm_share nlm_share;
	struct state_lock lock;
	struct state_deleg deleg;
	struct state_layout layout;
	struct state_9p_fid fid;
	uint32_t io_advise;
};
```

#### state_owner

这个指针自然就是指向拥有者了，此外还有一个指针`state_obj`，这是一个`fsal_obj_handle`类型的指针，由`state_owner和fsal_obj_handle`构成一个key映射到一个state上。

#### state_seqid

这是关于state上操作顺序的幂等性，open/close/lock/unlock操作都会使得这个序列号的增加。

#### 获得一个state

* 1

比如OP_OPEN操作中有这么一个接口：

```cpp
struct state_t *nfs4_State_Get_Obj(struct fsal_obj_handle *obj,
				   state_owner_t *owner);
```

它通过obj和对应的owner来形成一个key，并且找到对应的owner，这种是属于owner中已经存在state的情形。

* 2

也有当owner存在，但文件还未创建出来，自然没有对应的state，那么就通过export的预留操作，它被ganesha定义为：

```cpp
struct state_t *(*alloc_state)(struct fsal_export *exp_hdl,
				       enum state_type state_type,
				       struct state_t *related_state);
```

这里的type有很多种：

```cpp
enum state_type {
	STATE_TYPE_NONE = 0,
	STATE_TYPE_SHARE = 1,
	STATE_TYPE_DELEG = 2,
	STATE_TYPE_LOCK = 3,
	STATE_TYPE_LAYOUT = 4,
	STATE_TYPE_NLM_LOCK = 5,
	STATE_TYPE_NLM_SHARE = 6,
	STATE_TYPE_9P_FID = 7,
};
```

state在创建出来后通过`state_add_impl`进行初始化：

```cpp
state_status_t _state_add_impl(struct fsal_obj_handle *obj,
			       enum state_type state_type,
			       union state_data *state_data,
			       state_owner_t *owner_input, state_t **state,
			       struct state_refer *refer,
			       const char *func, int line);
```

初始化过程中比较关注的是state_seqid是设定为0，同时将各种链表链起来。

* 3

ganesha还提供了另外一种映射方式，它从`stateid_other`映射至state结构体，这样，如果客户端仅仅只需要进行read only操作，就可以不携带owner，只携带`stateid_other`即可，不过其实比较无所谓，毕竟state中同样可以找到owner。

注： __OP_OPEN在第一次创建出state结构体后，会在open返回之前将seqid设定为1，即使不是新创建的，同样会使得seqid增加，或者简单一点：OP_OPEN指令使seqid自增__。















