# ganesha mem layout

ExchangeID的时候，会生成一个record，这个record中有client_id

```cpp
struct nfs_client_record_t {
	int32_t cr_refcount;	
	pthread_mutex_t cr_mutex;	
	nfs_client_id_t *cr_confirmed_rec; 
	nfs_client_id_t *cr_unconfirmed_rec;	
	uint32_t cr_server_addr; 
	uint32_t cr_pnfs_flags; 
	int cr_client_val_len;  
	char cr_client_val[];   
};
```



```cpp
struct nfs_client_id_t {
	clientid4 cid_clientid;					// 这个结构体的标识符，可以供client快速获取这个结构体
	verifier4 cid_verifier;
	verifier4 cid_incoming_verifier;
	time_t cid_last_renew;					// lease
	nfs_clientid_confirm_state_t cid_confirmed; 
	bool cid_allow_reclaim;
	nfs_client_cred_t cid_credential;	
	char *cid_recov_tag;	
	nfs_client_record_t *cid_client_record;	 // 指向这个record
	struct glist_head cid_openowners;		 // openowner的链表头
	struct glist_head cid_lockowners;		 // lockowner的链表头
	pthread_mutex_t cid_mutex;	
	union {
		struct {
			struct rpc_call_channel cb_chan;
			gsh_addr_t cb_addr;
			uint32_t cb_callback_ident;
			char cb_client_r_addr[SOCK_NAME_MAX + 1];
			uint32_t cb_program;
			bool cb_chan_down;   
		} v40;		
		struct {
			bool cid_reclaim_complete; 
			struct glist_head cb_session_list;
		} v41;		
	} cid_cb;		
	time_t first_path_down_resp_time;  
	unsigned int cid_nb_session;	
	uint32_t cid_create_session_sequence;
	CREATE_SESSION4res cid_create_session_slot; 
	state_owner_t cid_owner;
	int32_t cid_refcount;	
	int cid_lease_reservations;	
	uint32_t cid_minorversion;
	uint32_t cid_stateid_counter;
	uint32_t curr_deleg_grants; 
	uint32_t num_revokes;       
	struct gsh_client *gsh_client;
};
```





## hash table



* ht_state_obj : 关于state_t这个结构体的哈希表，由于state_t 映射到 state_t
* ht_state_id: 关于



TODO:



