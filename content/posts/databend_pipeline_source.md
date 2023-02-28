---
title: "Databend pipeline executor notes"
date: 2022-05-23T20:57:38+08:00 
draft: false 
tags:
- 数据库
- DAG
- pipeline

---

# `Pipeline Executor`

## `Executor Tasks`

```rust
struct ExecutorTasks {
    tasks_size: usize,
    workers_sync_tasks: Vec<VecDeque<ProcessorPtr>>,
    workers_async_tasks: Vec<VecDeque<ProcessorPtr>>,
    workers_completed_async_tasks: Vec<VecDeque<CompletedAsyncTask>>,
}

pub fn create(workers_size: usize) -> ExecutorTasks {
        let mut workers_sync_tasks = Vec::with_capacity(workers_size);
        let mut workers_async_tasks = Vec::with_capacity(workers_size);
        let mut workers_completed_async_tasks = Vec::with_capacity(workers_size);

        for _index in 0..workers_size {
            workers_sync_tasks.push(VecDeque::new());
            workers_async_tasks.push(VecDeque::new());
            workers_completed_async_tasks.push(VecDeque::new());
        }

        ExecutorTasks {
            tasks_size: 0,
            workers_sync_tasks,
            workers_async_tasks,
            workers_completed_async_tasks,
        }
    }
```

`ExecutorTasks`按照`task`的类型，维护了每个`worker`的3种执行任务队列`VecDeque<ProcessorPtr>`，`worker`的编号使用`Vec`的`index`表示。

```rust
pub fn push_task(&mut self, worker_id: usize, task: ExecutorTask) {
        self.tasks_size += 1;
        debug_assert!(worker_id < self.workers_sync_tasks.len(), "out of index");
        let sync_queue = &mut self.workers_sync_tasks[worker_id];
        debug_assert!(worker_id < self.workers_async_tasks.len(), "out of index");
        let async_queue = &mut self.workers_async_tasks[worker_id];
        debug_assert!(
            worker_id < self.workers_completed_async_tasks.len(),
            "out of index"
        );
        let completed_queue = &mut self.workers_completed_async_tasks[worker_id];

        match task {
            ExecutorTask::None => unreachable!(),
            ExecutorTask::Sync(processor) => sync_queue.push_back(processor),
            ExecutorTask::Async(processor) => async_queue.push_back(processor),
            ExecutorTask::AsyncCompleted(task) => completed_queue.push_back(task),
        }
    }

pub fn best_worker_id(&self, mut worker_id: usize) -> usize {
        for _index in 0..self.workers_sync_tasks.len() {
            if worker_id >= self.workers_sync_tasks.len() {
                worker_id = 0;
            }

            if !self.workers_sync_tasks[worker_id].is_empty() {
                return worker_id;
            }

            if !self.workers_async_tasks[worker_id].is_empty() {
                return worker_id;
            }

            if !self.workers_completed_async_tasks[worker_id].is_empty() {
                return worker_id;
            }

            worker_id += 1;
        }

        worker_id
    }
```

- `push_task`按照任务任务类型将`task`放到`worker`的对应任务队列中。
- `best_worker_id`从指定的`work_id`顺序检索有待处理任务的`work_id`。

```rust
fn pop_worker_task(&mut self, worker_id: usize) -> ExecutorTask {
        if let Some(processor) = self.workers_sync_tasks[worker_id].pop_front() {
            return ExecutorTask::Sync(processor);
        }

        if let Some(task) = self.workers_completed_async_tasks[worker_id].pop_front() {
            return ExecutorTask::AsyncCompleted(task);
        }

        if let Some(processor) = self.workers_async_tasks[worker_id].pop_front() {
            return ExecutorTask::Async(processor);
        }

        ExecutorTask::None
    }

pub fn pop_task(&mut self, mut worker_id: usize) -> ExecutorTask {
        for _index in 0..self.workers_sync_tasks.len() {
            match self.pop_worker_task(worker_id) {
                ExecutorTask::None => {
                    worker_id += 1;
                    if worker_id >= self.workers_sync_tasks.len() {
                        worker_id = 0;
                    }
                }
                other => {
                    self.tasks_size -= 1;
                    return other;
                }
            }
        }

        ExecutorTask::None
    }
```

- `pop_worker_task`取出指定`worker_id`的任务或返回空。
- `pop_task`从指定的`worker_id`顺序检索任务或返回空。

```rust
pub struct ExecutorTasksQueue {
    finished: AtomicBool,
    workers_tasks: Mutex<ExecutorTasks>,
}

pub fn create(workers_size: usize) -> Arc<ExecutorTasksQueue> {
        Arc::new(ExecutorTasksQueue {
            finished: AtomicBool::new(false),
            workers_tasks: Mutex::new(ExecutorTasks::create(workers_size)),
        })
    }
```

任务队列包装了一个`ExecutorTasks`。

```rust
pub fn init_tasks(&self, mut tasks: VecDeque<ExecutorTask>) {
        let mut worker_id = 0;
        let mut workers_tasks = self.workers_tasks.lock();
        while let Some(task) = tasks.pop_front() {
            workers_tasks.push_task(worker_id, task);

            worker_id += 1;
            if worker_id == workers_tasks.workers_sync_tasks.len() {
                worker_id = 0;
            }
        }
    }
```

`init_tasks`方法将初始的任务，依次平均分配给每个`worker`，进入每个`worker`自己的相应队列。

```rust
pub fn push_tasks(&self, ctx: &mut ExecutorWorkerContext, mut tasks: VecDeque<ExecutorTask>) {
        let mut wake_worker_id = None;
        {
            let worker_id = ctx.get_worker_num();
            let mut workers_tasks = self.workers_tasks.lock();
            while let Some(task) = tasks.pop_front() {
                workers_tasks.push_task(worker_id, task);
            }

            wake_worker_id = Some(workers_tasks.best_worker_id(worker_id + 1));
        }

        if let Some(wake_worker_id) = wake_worker_id {
            ctx.get_workers_notify().wakeup(wake_worker_id);
        }
    }
```

`push_tasks`方法将任务加入某一个`worker`，同时尝试唤醒紧接着的一个`worker`。

```rust
pub fn steal_task_to_context(&self, context: &mut ExecutorWorkerContext) {
        let mut workers_tasks = self.workers_tasks.lock();

        if !workers_tasks.is_empty() {
            let task = workers_tasks.pop_task(context.get_worker_num());
            let is_async_task = matches!(&task, ExecutorTask::Async(_));

            context.set_task(task);

            let workers_notify = context.get_workers_notify();

            if is_async_task {
                workers_notify.inc_active_async_worker();
            }

            if !workers_tasks.is_empty() && !workers_notify.is_empty() {
                let worker_id = context.get_worker_num();
                let wakeup_worker_id = workers_tasks.best_worker_id(worker_id + 1);
                drop(workers_tasks);
                workers_notify.wakeup(wakeup_worker_id);
            }

            return;
        }

        // When tasks queue is empty and all workers are waiting, no new tasks will be generated.
        let workers_notify = context.get_workers_notify();
        if !workers_notify.has_waiting_async_task() && workers_notify.active_workers() <= 1 {
            drop(workers_tasks);
            self.finish();
            workers_notify.wakeup_all();
            return;
        }

        drop(workers_tasks);
        context.get_workers_notify().wait(context.get_worker_num());
    }
```

***steal_task_to_context***是一个关键方法，首先当还有待处理任务时，将任务拿到`context`对应的`worker`中，如果是异步任务，设置`inc_active_async_worker`
。然后再检查是否还有任务待处理，同时如果有空闲等待的`worker`，唤醒紧接当前`worker`的下一个`worker`。如果没有任务了，但是还有异步任务没有执行完或者还有其他其他的`worker`在执行中，当前`worker`
进入空闲等待状态，知道最后一个`worker`工作结束，设置`ExecutorTasksQueue`为完成，同时唤醒所有的`worker`。

## `ExecutorWorkerContext`

```rust
pub enum ExecutorTask {
    None,
    Sync(ProcessorPtr),
    Async(ProcessorPtr),
    // AsyncSchedule(ExecutingAsyncTask),
    AsyncCompleted(CompletedAsyncTask),
}

pub struct ExecutorWorkerContext {
    worker_num: usize,
    task: ExecutorTask,
    workers_notify: Arc<WorkersNotify>,
}
```

`ExecutorWorkerContext`是每个`worker`执行的上下文。

```rust
pub fn set_task(&mut self, task: ExecutorTask) {
        self.task = task
    }

pub fn take_task(&mut self) -> ExecutorTask {
        std::mem::replace(&mut self.task, ExecutorTask::None)
    }

pub unsafe fn execute_task(&mut self, exec: &PipelineExecutor) -> Result<Option<NodeIndex>> {
        match std::mem::replace(&mut self.task, ExecutorTask::None) {
            ExecutorTask::None => Err(ErrorCode::LogicalError("Execute none task.")),
            ExecutorTask::Sync(processor) => self.execute_sync_task(processor),
            ExecutorTask::Async(processor) => self.execute_async_task(processor, exec),
            ExecutorTask::AsyncCompleted(task) => match task.res {
                Ok(_) => Ok(Some(task.id)),
                Err(cause) => Err(cause),
            },
        }
    }

unsafe fn execute_sync_task(&mut self, processor: ProcessorPtr) -> Result<Option<NodeIndex>> {
        processor.process()?;
        Ok(Some(processor.id()))
    }

unsafe fn execute_async_task(
        &mut self,
        processor: ProcessorPtr,
        executor: &PipelineExecutor,
    ) -> Result<Option<NodeIndex>> {
        let worker_id = self.worker_num;
        let workers_notify = self.get_workers_notify().clone();
        let tasks_queue = executor.global_tasks_queue.clone();
        executor.async_runtime.spawn(async move {
            let res = processor.async_process().await;
            let task = CompletedAsyncTask::create(processor, worker_id, res);
            tasks_queue.completed_async_task(task);
            workers_notify.dec_active_async_worker();
            workers_notify.wakeup(worker_id);
        });

        Ok(None)
    }
```

核心的方法是`set_task`和`execute_task`。

- `set_task`为当前`worker`获取了一个任务。
- `execute_task`则是实际实际当前的任务，若任务是一个异步任务，则交由异步运行时实际执行，执行后将结果包装成`ExecutorTask::AsyncCompleted`任务，放入全局任务队列。同时唤醒`worker`
  ，如果当前`worker`在等待中则唤醒当前`worker`，如果当前`worker`没有在等待，则轮询等待中的`worker`，直到唤醒一个为止。

## `Executor notify`

```rust
struct WorkerNotify {
    waiting: Mutex<bool>,
    condvar: Condvar,
}

impl WorkerNotify {
    pub fn create() -> WorkerNotify {
        WorkerNotify {
            waiting: Mutex::new(false),
            condvar: Condvar::create(),
        }
    }
}

struct WorkersNotifyMutable {
    pub waiting_size: usize,
    pub workers_waiting: Vec<bool>,
}

pub struct WorkersNotify {
    workers: usize,
    waiting_async_task: AtomicUsize,
    mutable_state: Mutex<WorkersNotifyMutable>,
    workers_notify: Vec<WorkerNotify>,
}
```

- `WorkerNotify`构建一个条件变量，表示`worker`是否空闲等待。
- `WorkersNotify`用于保存哪些`worker`在空闲等待，通过`Vec`的`index`来关联指定的`worker`，同时记录在处理中的异步任务数。

```rust
pub fn wakeup(&self, worker_id: usize) {
        let mut mutable_state = self.mutable_state.lock();
        if mutable_state.waiting_size > 0 {
            mutable_state.waiting_size -= 1;

            if mutable_state.workers_waiting[worker_id] {
                mutable_state.workers_waiting[worker_id] = false;
                let mut waiting = self.workers_notify[worker_id].waiting.lock();

                *waiting = false;
                drop(mutable_state);
                self.workers_notify[worker_id].condvar.notify_one();
            } else {
                for (index, waiting) in mutable_state.workers_waiting.iter().enumerate() {
                    if *waiting {
                        mutable_state.workers_waiting[index] = false;
                        let mut waiting = self.workers_notify[index].waiting.lock();

                        *waiting = false;
                        drop(mutable_state);
                        self.workers_notify[index].condvar.notify_one();
                        return;
                    }
                }
            }
        }
    }

pub fn wakeup_all(&self) {
        let mut mutable_state = self.mutable_state.lock();
        if mutable_state.waiting_size > 0 {
            mutable_state.waiting_size = 0;

            for index in 0..mutable_state.workers_waiting.len() {
                mutable_state.workers_waiting[index] = false;
            }

            drop(mutable_state);
            for index in 0..self.workers_notify.len() {
                let mut waiting = self.workers_notify[index].waiting.lock();

                if *waiting {
                    *waiting = false;
                    self.workers_notify[index].condvar.notify_one();
                }
            }
        }
    }

pub fn wait(&self, worker_id: usize) {
        let mut mutable_state = self.mutable_state.lock();
        mutable_state.waiting_size += 1;
        mutable_state.workers_waiting[worker_id] = true;
        let mut waiting = self.workers_notify[worker_id].waiting.lock();

        *waiting = true;
        drop(mutable_state);
        self.workers_notify[worker_id].condvar.wait(&mut waiting);
    }
```

- `wakeup`唤醒空闲等待的`worker`，如果指定的`worker`已经被唤醒，则轮询等待中的`worker`，知道唤醒一个位置。
- `wakeup_all`只要有空闲等待的`worker`则全部唤醒。
- `wait`使指定的`worker`空闲等待。

## `Port`

```rust
const HAS_DATA: usize = 0b1;
const NEED_DATA: usize = 0b10;
const IS_FINISHED: usize = 0b100;

const FLAGS_MASK: usize = 0b111;
const UNSET_FLAGS_MASK: usize = !FLAGS_MASK;

#[repr(align(8))]
pub struct SharedData(pub Result<DataBlock>);

pub struct SharedStatus {
    data: AtomicPtr<SharedData>,
}
```

`port`中核心的数据结构是`SharedStatus`，使用原子指针来在多线程中共享`SharedData`，`SharedData`是一个`DataBlock`。

```rust
pub fn swap(
        &self,
        data: *mut SharedData,
        set_flags: usize,
        unset_flags: usize,
    ) -> *mut SharedData {
        let mut expected = std::ptr::null_mut();
        let mut desired = (data as usize | set_flags) as *mut SharedData;

        loop {
            match self.data.compare_exchange_weak(
                expected,
                desired,
                Ordering::SeqCst,
                Ordering::Relaxed,
            ) {
                Err(new_expected) => {
                    expected = new_expected;
                    let address = expected as usize;
                    let desired_data = desired as usize & UNSET_FLAGS_MASK;
                    let desired_flags = (address & FLAGS_MASK & !unset_flags) | set_flags;
                    desired = (desired_data | desired_flags) as *mut SharedData;
                }
                Ok(old_value) => {
                    let old_value_ptr = old_value as usize;
                    return (old_value_ptr & UNSET_FLAGS_MASK) as *mut SharedData;
                }
            }
        }
    }

pub fn set_flags(&self, set_flags: usize, unset_flags: usize) -> usize {
        let mut expected = std::ptr::null_mut();
        let mut desired = set_flags as *mut SharedData;
        loop {
            match self.data.compare_exchange_weak(
                expected,
                desired,
                Ordering::SeqCst,
                Ordering::Relaxed,
            ) {
                Ok(old_value) => {
                    return old_value as usize & FLAGS_MASK;
                }
                Err(new_expected) => {
                    expected = new_expected;
                    let address = expected as usize;
                    let desired_data = address & UNSET_FLAGS_MASK;
                    let desired_flags = (address & FLAGS_MASK & !unset_flags) | set_flags;
                    desired = (desired_data | desired_flags) as *mut SharedData;
                }
            }
        }
    }

    #[inline(always)]
    pub fn get_flags(&self) -> usize {
        self.data.load(Ordering::Relaxed) as usize & FLAGS_MASK
    }
```

`port`中使用交换指针地址而非实际数据来减少内存的复制，同时复用指针地址的低三位来标识状态，实际的指针地址和状态可以通过`mask`来得出实际的值。

```rust
pub struct InputPort {
    shared: UnSafeCellWrap<Arc<SharedStatus>>,
    update_trigger: UnSafeCellWrap<*mut UpdateTrigger>,
}

pub fn pull_data(&self) -> Option<Result<DataBlock>> {
        unsafe {
            UpdateTrigger::update_input(&self.update_trigger);
            let unset_flags = HAS_DATA | NEED_DATA;
            match self.shared.swap(std::ptr::null_mut(), 0, unset_flags) {
                address if address.is_null() => None,
                address => Some((*Box::from_raw(address)).0),
            }
        }
    }

pub fn set_need_data(&self) {
        unsafe {
            let flags = self.shared.set_flags(NEED_DATA, NEED_DATA);
            if flags & NEED_DATA == 0 {
                UpdateTrigger::update_input(&self.update_trigger);
            }
        }
    }

pub fn finish(&self) {
        unsafe {
            let flags = self.shared.set_flags(IS_FINISHED, IS_FINISHED);

            if flags & IS_FINISHED == 0 {
                UpdateTrigger::update_input(&self.update_trigger);
            }
        }
    }

pub struct OutputPort {
    shared: UnSafeCellWrap<Arc<SharedStatus>>,
    update_trigger: UnSafeCellWrap<*mut UpdateTrigger>,
}

pub fn push_data(&self, data: Result<DataBlock>) {
        unsafe {
            UpdateTrigger::update_output(&self.update_trigger);

            let data = Box::into_raw(Box::new(SharedData(data)));
            self.shared.swap(data, HAS_DATA, HAS_DATA);
        }
    }


pub fn finish(&self) {
        unsafe {
            let flags = self.shared.set_flags(IS_FINISHED, IS_FINISHED);

            if flags & IS_FINISHED == 0 {
                UpdateTrigger::update_output(&self.update_trigger);
            }
        }
    }

pub unsafe fn connect(input: &InputPort, output: &OutputPort) {
    let shared_status = SharedStatus::create();

    input.set_shared(shared_status.clone());
    output.set_shared(shared_status);
}
```

- `pull_data`方法会改变`input_port`中的数据，`set_need_data`和`finish`会改变`port`的状态，三个方法都会触发`trigger`，实际会`push`
  一个`DirectedEdge::Target`到全局的`updated_edges`中。
- `push_data`方法会改变`output_port`中的数据，`finish`会改变`port`的状态，两个方法都会触发`trigger`,实际会`push`一个`DirectedEdge::Source`
  到全局的`updated_edges`中。
- `connect`是一个`input_port`和一个`output_port`可以共享同一个`SharedStatus`。

## `Execute Graph`

整个`pipeline`的`pipes`会循环迭代所有的`processor`和`input_port`,`output_port`，构建成`graph`的`node`和`edge`。同时在`node`上会创建`input_port`
和`output_port`的触发器，触发器会触发`edge`被调度，而后就会调度到原节点的上下游节点。

```rust
    pub fn create(pipeline: NewPipeline) -> Result<ExecutingGraph> {
        // let (nodes_size, edges_size) = pipeline.graph_size();
        let mut graph = StableGraph::new();

        let mut node_stack = Vec::new();
        let mut edge_stack: Vec<Arc<OutputPort>> = Vec::new();
        for query_pipe in &pipeline.pipes {
            match query_pipe {
                NewPipe::ResizePipe {
                    processor,
                    inputs_port,
                    outputs_port,
                } => unsafe {
                    assert_eq!(node_stack.len(), inputs_port.len());

                    let resize_node = Node::create(processor, inputs_port, outputs_port);
                    let target_index = graph.add_node(resize_node.clone());
                    processor.set_id(target_index);

                    for index in 0..node_stack.len() {
                        let source_index = node_stack[index];
                        let edge_index = graph.add_edge(source_index, target_index, ());

                        let input_trigger = resize_node.create_trigger(edge_index);
                        inputs_port[index].set_trigger(input_trigger);
                        edge_stack[index]
                            .set_trigger(graph[source_index].create_trigger(edge_index));
                        connect(&inputs_port[index], &edge_stack[index]);
                    }

                    node_stack.clear();
                    edge_stack.clear();
                    for output_port in outputs_port {
                        node_stack.push(target_index);
                        edge_stack.push(output_port.clone());
                    }
                },
                NewPipe::SimplePipe {
                    processors,
                    inputs_port,
                    outputs_port,
                } => unsafe {
                    assert_eq!(node_stack.len(), inputs_port.len());
                    assert!(inputs_port.is_empty() || inputs_port.len() == processors.len());
                    assert!(outputs_port.is_empty() || outputs_port.len() == processors.len());

                    let mut new_node_stack = Vec::with_capacity(outputs_port.len());
                    let mut new_edge_stack = Vec::with_capacity(outputs_port.len());

                    for index in 0..processors.len() {
                        let mut p_inputs_port = Vec::with_capacity(1);
                        let mut p_outputs_port = Vec::with_capacity(1);

                        if !inputs_port.is_empty() {
                            p_inputs_port.push(inputs_port[index].clone());
                        }

                        if !outputs_port.is_empty() {
                            p_outputs_port.push(outputs_port[index].clone());
                        }

                        let target_node =
                            Node::create(&processors[index], &p_inputs_port, &p_outputs_port);
                        let target_index = graph.add_node(target_node.clone());
                        processors[index].set_id(target_index);

                        if !node_stack.is_empty() {
                            let source_index = node_stack[index];
                            let edge_index = graph.add_edge(source_index, target_index, ());

                            inputs_port[index].set_trigger(target_node.create_trigger(edge_index));
                            edge_stack[index]
                                .set_trigger(graph[source_index].create_trigger(edge_index));
                            connect(&inputs_port[index], &edge_stack[index]);
                        }

                        if !outputs_port.is_empty() {
                            new_node_stack.push(target_index);
                            new_edge_stack.push(outputs_port[index].clone());
                        }
                    }

                    node_stack = new_node_stack;
                    edge_stack = new_edge_stack;
                },
            };
        }

        // Assert no output.
        assert_eq!(node_stack.len(), 0);
        Ok(ExecutingGraph { graph })
    }
```
