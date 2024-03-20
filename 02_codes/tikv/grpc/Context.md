
#kvrpcpb_Context

```rust
#[derive(PartialEq,Clone,Default)]  
pub struct Context {  
// message fields  
	pub region_id: u64,  
	pub region_epoch: ::protobuf::SingularPtrField<super::metapb::RegionEpoch>,  
	pub peer: ::protobuf::SingularPtrField<super::metapb::Peer>,  
	pub term: u64,  
	pub priority: CommandPri,  
	pub isolation_level: IsolationLevel,  
	pub not_fill_cache: bool,  
	pub sync_log: bool,  
	pub record_time_stat: bool,  
	pub record_scan_stat: bool,  
	pub replica_read: bool,  
	pub resolved_locks: ::std::vec::Vec<u64>,  
	pub max_execution_duration_ms: u64,  
	pub applied_index: u64,  
	pub task_id: u64,  
	pub stale_read: bool,  
	pub resource_group_tag: ::std::vec::Vec<u8>,  
	pub disk_full_opt: DiskFullOpt,  
	pub is_retry_request: bool,  
	pub api_version: ApiVersion,  
	pub committed_locks: ::std::vec::Vec<u64>,  
	pub trace_context: ::protobuf::SingularPtrField<super::tracepb::TraceContext>,  
	pub request_source: ::std::string::String,  
	pub txn_source: u64,  
	pub busy_threshold_ms: u32,  
	pub resource_control_context: ::protobuf::SingularPtrField<ResourceControlContext>,  
	pub keyspace_id: u32,  
	pub buckets_version: u64,  
	pub source_stmt: ::protobuf::SingularPtrField<SourceStmt>,  
	// special fields  
	pub unknown_fields: ::protobuf::UnknownFields,  
	pub cached_size: ::protobuf::CachedSize,  
}
```

问题:
1.  java客户端的kvrpcpb.proto 中没有request_source属性

---


#tikv-grpc-ResourceControlContext
```rust
#[derive(PartialEq,Clone,Default)]  
pub struct ResourceControlContext {  
	// message fields  
	pub resource_group_name: ::std::string::String,  
	pub penalty: ::protobuf::SingularPtrField<super::resource_manager::Consumption>,  
	pub override_priority: u64,  
	// special fields  
	pub unknown_fields: ::protobuf::UnknownFields,  
	pub cached_size: ::protobuf::CachedSize,  
}
```