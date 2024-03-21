https://cn.pingcap.com/blog/tikv-source-code-reading-read/

推荐tests/integrations/server/kv_server.rs 单测

生成处理请求函数 
```rust
handle_request!(kv_get, future_get, GetRequest, GetResponse, has_time_detail);
```

--- 
宏实现 #handle_request！
```rust
macro_rules! handle_request {  
	
	($fn_name: ident, $future_name: ident, $req_ty: ident, $resp_ty: ident) => { 
		handle_request!($fn_name, $future_name, $req_ty, $resp_ty, no_time_detail);  
	};  
	//它定义了一个函数 `$fn_name`，该函数接受一系列参数，并生成相应的代码块。
	($fn_name: ident, $future_name: ident, $req_ty: ident, $resp_ty: ident, $time_detail: tt) => {  
	//$fn_name 函数名
	//$future_name
	//$req_ty
	//$resp_ty
	//$time_detail
	//self #kv_Service 
	fn $fn_name(&mut self, ctx: RpcContext<'_>, req: $req_ty, sink: UnarySink<$resp_ty>) {  
	//执行proxy 代理访问到leader
	forward_unary!(self.proxy, $fn_name, ctx, req, sink);
	//获得当前时间  
	let begin_instant = Instant::now();  
	  
	let source = req.get_context().get_request_source().to_owned();  
	let resource_control_ctx = req.get_context().get_resource_control_context();  
	if let Some(resource_manager) = &self.resource_manager {  
		resource_manager.consume_penalty(resource_control_ctx);  
	}  
	GRPC_RESOURCE_GROUP_COUNTER_VEC  
	.with_label_values(&[resource_control_ctx.get_resource_group_name(), resource_control_ctx.get_resource_group_name()])  
	.inc();  
	let resp = $future_name(&self.storage, req);  //run this
	let task = async move {  
		let resp = resp.await?;  
		//耗时
		let elapsed = begin_instant.saturating_elapsed();  
		//总时间
		set_total_time!(resp, elapsed, $time_detail);
		//发送resp  
		sink.success(resp).await?;  
		GRPC_MSG_HISTOGRAM_STATIC  
			.$fn_name  
			.observe(elapsed.as_secs_f64()); 
		//统计命令 耗时和次数	 
		record_request_source_metrics(source, elapsed);  
		ServerResult::Ok(())  
	}  
	.map_err(|e| {  
		log_net_error!(e, "kv rpc failed";  
			"request" => stringify!($fn_name)  
		);  
		//失败统计
		GRPC_MSG_FAIL_COUNTER.$fn_name.inc();  
	})  
	.map(|_|());  
			  
			ctx.spawn(task);  
		}  
	}  
}
```

#kv_Service
```rust
pub struct Service<E: Engine, L: LockManager, F: KvFormat> {  
	store_id: u64,  
	/// Used to handle requests related to GC.  
	// TODO: make it Some after GC is supported for v2.  
	gc_worker: GcWorker<E>,  
	// For handling KV requests.  
	storage: Storage<E, L, F>,  
	// For handling coprocessor requests.  
	copr: Endpoint<E>,  
	// For handling corprocessor v2 requests.  
	copr_v2: coprocessor_v2::Endpoint,  
	// For handling snapshot.  
	snap_scheduler: Scheduler<SnapTask>,  
	// For handling `CheckLeader` request.  
	check_leader_scheduler: Scheduler<CheckLeaderTask>,  
	  
	enable_req_batch: bool,  
	  
	grpc_thread_load: Arc<ThreadLoadPool>,  
	  
	proxy: Proxy,  
	  
	// Go `server::Config` to get more details.  
	reject_messages_on_memory_ratio: f64,  
	  
	resource_manager: Option<Arc<ResourceGroupManager>>,  
}
```
首先调用forward_unary!宏

```rust
#[macro_export]  
macro_rules! forward_unary {  
	($proxy:expr, $func:ident, $ctx:ident, $req:ident, $resp:ident) => {{ 
		//header 里面有tikv-forwarded-host value就是需要代理的
		let addr = $crate::server::get_target_address(&$ctx);  
		if !addr.is_empty() {  
			$ctx.spawn($proxy.call_on(addr, move |client| {  
				let f = paste::paste! {  
					client.[<$func _async>](&$req).unwrap()  
				};  
				client.spawn(async move {  
					let res = match f.await {  
						Ok(r) => $resp.success(r).await,  
						Err(grpcio::Error::RpcFailure(r)) => $resp.fail(r).await,  
						Err(e) => Err(e),  
					};  
					match res {  
						Ok(()) => GRPC_PROXY_MSG_COUNTER.$func.success.inc(),  
						Err(e) => {  
							debug!("forward kv rpc failed";  
								"request" => stringify!($func),  
								"err" => ?e  
							);  
						GRPC_PROXY_MSG_COUNTER.$func.fail.inc();  
						}  
					}  
				})  
			}));  
			return;  
		}  
	}};  
}
```


这是一个 Rust 宏，用于将一个 gRPC 请求转发到另一个 gRPC 服务。它接受五个参数：`$proxy` 表示一个 gRPC 代理对象，`$func` 表示要调用的 gRPC 方法名，`$ctx` 表示 gRPC 上下文对象，`$req` 表示 gRPC 请求对象，`$resp` 表示 gRPC 响应对象。  
  
宏的作用是将 `$proxy` 对象调用 `$func` 方法，并将 `$req` 请求对象作为参数传递给该方法。如果调用成功，宏会将返回值包装成一个 gRPC 成功响应，并将其写入 `$resp` 响应对象中。如果调用失败，宏会将错误信息包装成一个 gRPC 失败响应，并将其写入 `$resp` 响应对象中。  
  
在宏展开的过程中，使用了一些 Rust 的高级特性，例如 `paste` crate 中的 `paste!` 宏，用于动态生成方法名。另外，宏中还使用了 `$crate` 变量，用于引用当前 crate 的名称，以使宏更加通用和可移植。  
  
总之，这个宏是一个实用的工具，可以帮助 Rust 开发者更方便地进行 gRPC 开发

---
函数执行
```rust
fn future_get<E: Engine, L: LockManager, F: KvFormat>(  
	storage: &Storage<E, L, F>,  
	mut req: GetRequest,  
) -> impl Future<Output = ServerResult<GetResponse>> {  
		//在全局的 `GLOBAL_TRACKERS` 中插入一个新的 `Tracker` 对象，用于跟踪请求的信息。`Tracker` 是一个用于跟踪请求和性能统计的结构体。
		let tracker = GLOBAL_TRACKERS.insert(Tracker::new(RequestInfo::new(  
			req.get_context(),  
			RequestType::KvGet,  
			req.get_version(),  
		)));  
		//将当前线程的跟踪器令牌设置为新插入的 `Tracker` 对象。
		set_tls_tracker_token(tracker); 
		//当前时间 
		let start = Instant::now();  
		let v = storage.get(  
			req.take_context(),  
			Key::from_raw(req.get_key()),  
			req.get_version().into(),  
		);  
	  
		async move {  
			let v = v.await;  
			//耗时
			let duration = start.saturating_elapsed();  
			let mut resp = GetResponse::default(); 
			//过滤出region 相关error 
			if let Some(err) = extract_region_error(&v) {  
				resp.set_region_error(err);  
			} else {  
				match v {  
					Ok((val, stats)) => {  
						//更新状态
						let exec_detail_v2 = resp.mut_exec_details_v2();  
						let scan_detail_v2 = exec_detail_v2.mut_scan_detail_v2();  
						stats.stats.write_scan_detail(scan_detail_v2);  
						GLOBAL_TRACKERS.with_tracker(tracker, |tracker| {  
							tracker.write_scan_detail(scan_detail_v2);  
						});  
						set_time_detail(exec_detail_v2, duration, &stats.latency_stats);  
						match val {  
							Some(val) => resp.set_value(val),  
							None => resp.set_not_found(true),  
						}  
					}  
					Err(e) => resp.set_error(extract_key_error(&e)),  
				}  
			}  
		//移除tracker	
		GLOBAL_TRACKERS.remove(tracker);  
		Ok(resp)  
	}  
}
```


---

in src/storeage/mod.rs

```rust
pub fn get(  
&self,  
	mut ctx: Context,  
	key: Key,  
	start_ts: TimeStamp,  
) -> impl Future<Output = Result<(Option<Value>, KvGetStatistics)>> { 
	//记录开始时间
	let stage_begin_ts = Instant::now();  
	//获得超时时间
	let deadline = Self::get_deadline(&ctx);  
	const CMD: CommandKind = CommandKind::get;  
	let priority = ctx.get_priority();  
	//从上下文中获取任务元数据
	let metadata = TaskMetadata::from_ctx(ctx.get_resource_control_context());  
	let resource_limiter = self.resource_manager.as_ref().and_then(|r| {  
		r.get_resource_limiter(  
			ctx.get_resource_control_context().get_resource_group_name(),  
			ctx.get_request_source(),  
			ctx.get_resource_control_context().get_override_priority(),  
		)  
	});  
	//
	let priority_tag = get_priority_tag(priority);  
	
	let resource_tag = self.resource_tag_factory.new_tag_with_key_ranges(  
		&ctx,  
		vec![(key.as_encoded().to_vec(), key.as_encoded().to_vec())],  
	);  
	let concurrency_manager = self.concurrency_manager.clone();  
	let api_version = self.api_version;  
	let busy_threshold = Duration::from_millis(ctx.f as u64);  
	  
	let quota_limiter = self.quota_limiter.clone();  
	let mut sample = quota_limiter.new_sample(true);  
	  
	self.read_pool_spawn_with_busy_check(  
	busy_threshold,  
	async move {  
		let stage_scheduled_ts = Instant::now();  
		tls_collect_query(  
		ctx.get_region_id(),  
		ctx.get_peer(),  
		key.as_encoded(),  
		key.as_encoded(),  
		false,  
		QueryKind::Get,  
	);  
	  
	KV_COMMAND_COUNTER_VEC_STATIC.get(CMD).inc();  
	SCHED_COMMANDS_PRI_COUNTER_VEC_STATIC  
	.get(priority_tag)  
	.inc();  
	  
	deadline.check()?;  
	  
	Self::check_api_version(api_version, ctx.api_version, CMD, [key.as_encoded()])?;  
	  
	let command_duration = Instant::now();  
	  
	// The bypass_locks and access_locks set will be checked at most once.  
	// `TsSet::vec` is more efficient here.  
	let bypass_locks = TsSet::vec_from_u64s(ctx.take_resolved_locks());  
	let access_locks = TsSet::vec_from_u64s(ctx.take_committed_locks());  
	  
	let snap_ctx = prepare_snap_ctx(  
	&ctx,  
	iter::once(&key),  
	start_ts,  
	&bypass_locks,  
	&concurrency_manager,  
	CMD,  
	)?;  
	let snapshot =  
	Self::with_tls_engine(|engine| Self::snapshot(engine, snap_ctx)).await?;  
	  
	{  
	deadline.check()?;  
	let begin_instant = Instant::now();  
	let stage_snap_recv_ts = begin_instant;  
	let buckets = snapshot.ext().get_buckets();  
	let mut statistics = Statistics::default();  
	let result = Self::with_perf_context(CMD, || {  
	let _guard = sample.observe_cpu();  
	let snap_store = SnapshotStore::new(  
	snapshot,  
	start_ts,  
	ctx.get_isolation_level(),  
	!ctx.get_not_fill_cache(),  
	bypass_locks,  
	access_locks,  
	false,  
	);  
	snap_store  
	.get(&key, &mut statistics)  
	// map storage::txn::Error -> storage::Error  
	.map_err(Error::from)  
	.map(|r| {  
	KV_COMMAND_KEYREAD_HISTOGRAM_STATIC.get(CMD).observe(1_f64);  
	r  
	})  
	});  
	metrics::tls_collect_scan_details(CMD, &statistics);  
	metrics::tls_collect_read_flow(  
	ctx.get_region_id(),  
	Some(key.as_encoded()),  
	Some(key.as_encoded()),  
	&statistics,  
	buckets.as_ref(),  
	);  
	let now = Instant::now();  
	SCHED_PROCESSING_READ_HISTOGRAM_STATIC  
	.get(CMD)  
	.observe(duration_to_sec(  
	now.saturating_duration_since(begin_instant),  
	));  
	SCHED_HISTOGRAM_VEC_STATIC.get(CMD).observe(duration_to_sec(  
	now.saturating_duration_since(command_duration),  
	));  
	  
	let read_bytes = key.len()  
	+ result  
	.as_ref()  
	.unwrap_or(&None)  
	.as_ref()  
	.map_or(0, |v| v.len());  
	sample.add_read_bytes(read_bytes);  
	let quota_delay = quota_limiter.consume_sample(sample, true).await;  
	if !quota_delay.is_zero() {  
	TXN_COMMAND_THROTTLE_TIME_COUNTER_VEC_STATIC  
	.get(CMD)  
	.inc_by(quota_delay.as_micros() as u64);  
	}  
	  
	let stage_finished_ts = Instant::now();  
	let schedule_wait_time =  
	stage_scheduled_ts.saturating_duration_since(stage_begin_ts);  
	let snapshot_wait_time =  
	stage_snap_recv_ts.saturating_duration_since(stage_scheduled_ts);  
	let wait_wall_time =  
	stage_snap_recv_ts.saturating_duration_since(stage_begin_ts);  
	let process_wall_time =  
	stage_finished_ts.saturating_duration_since(stage_snap_recv_ts);  
	let latency_stats = StageLatencyStats {  
	schedule_wait_time_ns: schedule_wait_time.as_nanos() as u64,  
	snapshot_wait_time_ns: snapshot_wait_time.as_nanos() as u64,  
	wait_wall_time_ns: wait_wall_time.as_nanos() as u64,  
	process_wall_time_ns: process_wall_time.as_nanos() as u64,  
	};  
	with_tls_tracker(|tracker| {  
	tracker.metrics.read_pool_schedule_wait_nanos =  
	schedule_wait_time.as_nanos() as u64;  
	});  
	Ok((  
	result?,  
	KvGetStatistics {  
	stats: statistics,  
	latency_stats,  
	},  
	))  
	}  
	}  
	.in_resource_metering_tag(resource_tag),  
	priority,  
	thread_rng().next_u64(),  
	metadata,  
	resource_limiter,  
	)  
}
```

这段代码是一个公共方法 `get`，它接受一个 `Context` 对象、一个 `Key` 对象和一个 `start_ts` 参数，并返回一个实现了 `Future` trait 的类型，其输出是一个 `Result<(Option<Value>, KvGetStatistics)>`。  
  
函数的主要作用是执行一个异步的获取操作，从存储引擎中获取指定键的值，并返回一个包含获取结果和统计信息的元组。  
  
以下是函数的主要步骤：  
  
1. 记录获取操作开始的时间点 `stage_begin_ts`。  
  
2. 获取请求的截止时间 `deadline`。  
  
3. 定义常量 `CMD`，表示当前操作的命令类型为 `get`。  
  
4. 获取请求的优先级 `priority`。  
  
5. 从上下文中获取任务元数据 `metadata`。  
  
6. 如果存在资源管理器 `resource_manager`，则获取与请求相关的资源限制器 `resource_limiter`。  
  
7. 使用优先级创建一个资源标签 `priority_tag`。  
  
8. 使用键范围创建一个资源标签 `resource_tag`。  
  
9. 克隆并获取并发管理器 `concurrency_manager`。  
  
10. 获取存储引擎的 API 版本。  
  
11. 获取繁忙阈值 `busy_threshold`，将其转换为 `Duration` 类型。  
  
12. 克隆并获取配额限制器 `quota_limiter`。  
  
13. 创建一个配额限制器的采样对象 `sample`。  
  
14. 在读取线程池中执行异步闭包，同时检查是否超过繁忙阈值。  
  
15. 在异步闭包中执行以下操作：  
- 记录调度开始的时间点 `stage_scheduled_ts`。  
- 收集查询的跟踪信息。  
- 增加 `CMD` 命令的计数器。  
- 增加调度命令的优先级计数器。  
- 检查请求是否超过截止时间。  
- 检查 API 版本是否匹配。  
- 记录命令执行的时间点 `command_duration`。  
- 解析已解决的锁和已提交的锁。  
- 准备快照上下文，并创建快照。  
- 在快照上下文中执行以下操作：  
- 检查截止时间。  
- 记录快照接收的时间点 `stage_snap_recv_ts`。  
- 获取快照的桶。  
- 创建统计信息对象。  
- 使用性能上下文执行以下操作：  
- 使用配额限制器的采样对象观察 CPU 使用情况。  
- 创建快照存储对象，并获取键的值和统计信息。  
- 更新键读取的直方图。  
- 收集扫描详情的指标。  
- 收集读取流量的指标。  
- 记录处理读取命令的时间。  
- 记录命令处理的时间。  
- 计算读取的字节数，并将其添加到采样对象中。  
- 使用配额限制器消费采样对象，并获取配额延迟。  
- 如果配额延迟不为零，则增加配额限制器的计数器。  
- 记录调度完成的时间点 `stage_finished_ts`。  
- 计算调度等待时间、快照等待时间、总等待时间和处理时间，并创建阶段延迟统计信息。  
- 更新跟踪器的读取线程池调度等待时间。  
- 返回获取操作的结果和统计信息的元组。  
  
16. 在资源计量标签中执行异步闭包，同时传递优先级、随机数和元数据等参数。  
  
总体来说，这个方法主要是为了执行异步的获取操作，并根据操作的结果构建一个包含获取结果和统计信息的元组。它还使用了资源标签、并发管理器和配额限制器来进行资源管理和配额控制。

---