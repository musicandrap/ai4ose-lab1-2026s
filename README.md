# 3.14项目进度文档

## 阶段一1

完成了一阶段的ch1-8的基础练习，通过了**全部**的测试

## 阶段二：T2L5————同步互斥机制（从“能跑”到“公平/可证明不饿死”）

**1.多个锁的初步设计**

​	**SpinLock实现：**

	结构：
	pub struct SpinLock {
		locked: AtomicBool,
		stats: LockStats,
	}
	上锁：
	pub fn lock(&self) -> SpinLockGuard<'_> {
	    let start_time = current_time_ms();
	    let mut contended = false;
	
	    #[cfg(target_arch = "riscv64")]
	    let enabled = unsafe { riscv::register::sstatus::read().sie() };
	
	    #[cfg(target_arch = "riscv64")]
	    unsafe {
	        riscv::register::sstatus::clear_sie()//禁止中断
	    };
	
	    loop {
	        if self
	            .locked
	            .compare_exchange(false, true, Ordering::Acquire, Ordering::Relaxed)
	            .is_ok()
	        {
	            break;
	        }
	
	        if !contended {
	            contended = true;
	            self.stats.record_contention();
	        }
	
	        core::hint::spin_loop();//自旋
	    }
	
	    if contended {
	        let wait_time = current_time_ms() - start_time;
	        self.stats.record_wait_time(wait_time);
	    }
	
	    SpinLockGuard {
	        lock: self,
	        start_time: current_time_ms(),
	        #[cfg(target_arch = "riscv64")]
	        enabled,
	    }
	}
​	**Mutex：实现**

```
结构：
pub struct MutexInner {
    pub locked: bool,
    pub holder: Option<ProcId>,
    pub wait_queue: VecDeque<ProcId>,
}
pub struct Mutex {
    locked: AtomicBool,
    inner: UnsafeCell<MutexInner>,
    stats: LockStats,
}
上锁：
pub fn lock(&self, pid: ProcId) -> (bool, Option<ProcId>) {
        let start_time = current_time_ms();

        #[cfg(target_arch = "riscv64")]
        let enabled = unsafe { riscv::register::sstatus::read().sie() };

        #[cfg(target_arch = "riscv64")]
        unsafe {
            riscv::register::sstatus::clear_sie()
        };

        loop {
            if self
                .locked
                .compare_exchange(false, true, Ordering::Acquire, Ordering::Relaxed)
                .is_ok()
            {
                break;
            }
            core::hint::spin_loop();
        }

        let inner = unsafe { &mut *self.inner.get() };

        if inner.locked {
            inner.wait_queue.push_back(pid);//交给调度器，"睡眠"
            self.stats.record_contention();
            let wait_time = current_time_ms() - start_time;
            self.stats.record_wait_time(wait_time);

            #[cfg(target_arch = "riscv64")]
            if enabled {
                unsafe {
                    riscv::register::sstatus::set_sie()
                };
            }

            (false, None)
        } else {
            inner.locked = true;
            inner.holder = Some(pid);

            #[cfg(target_arch = "riscv64")]
            if enabled {
                unsafe {
                    riscv::register::sstatus::set_sie()
                };
            }

            (true, None)
        }
    }
```

​	**Semaphore**

```
pub struct SemaphoreInner {
    pub count: isize,                 // 可用资源数量
    pub wait_queue: VecDeque<ProcId>, // 等待队列
}

pub struct Semaphore {
    locked: AtomicBool,               // 自旋锁，用于保护内部数据
    inner: UnsafeCell<SemaphoreInner>,
}
以down申请资源为例：
fn down(pid: ProcId) -> bool {

    acquire_spinlock()

    inner.count -= 1 //申请一块资源

    if inner.count < 0 {
        inner.wait_queue.push_back(pid)

        release_spinlock()

        return false      // 当前进程需要睡眠
    }

    release_spinlock()

    return true           // 获得资源
}
```

​	**Condvar： **

```
pub struct CondvarInner {
    pub wait_queue: VecDeque<ProcId>,
}

pub struct Condvar {
    locked: AtomicBool,
    inner: UnsafeCell<CondvarInner>,
}
以wait等待为例：
执行流程：

将当前线程加入等待队列

释放 mutex

当前线程睡眠

被唤醒后重新获取 mutex
fn wait(pid, mutex):

    acquire_spinlock()

    wait_queue.push_back(pid)

    release_spinlock()

    mutex.unlock() //这里不释放的话会导致这个资源永远取不到了，直接g

    sleep(pid)

    mutex.lock()
```

​	**性能设计：**

```
#[derive(Debug, Default)]
pub struct LockStats {
    pub contentions: AtomicUsize,
    pub total_hold_time: AtomicUsize,
    pub max_hold_time: AtomicUsize,
    pub total_wait_time: AtomicUsize,
    pub max_wait_time: AtomicUsize,
}
```

