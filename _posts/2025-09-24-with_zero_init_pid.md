---
title: docker 容器内的 init 进程以及 SIGNAL_UNKILLABLE
date: 2025-09-24 21:40:00 +0800
categories: [Blogging, docker, linux]
tags: [writing]
---

有点杂，记录一下glog在容器环境下fatal log导致hang死的情况

### 容器内的PID 1

首先知道的是，pid = 1的进程，在linux里是很特殊的，Linux Kernel会忽略掉通过kill()/tgkill()系统调用发送给PID为1的进程的异常信号(SIGSEGV, SIGABRT, SIGBUS等)

而glog里的，初始化会为fatal signal注册一个handler，然后回调用InvokeDefaultSignalHandler

```cpp
void InstallFailureSignalHandler() 
void FailureSignalHandler
void InvokeDefaultSignalHandler(int signal_number)
```

比较搞的是在FailureSignalHandler里，调用kill之后还有一坨代码

```cpp
if (old_thread_id_pointer != nullptr) {
  // We've already entered the signal handler.  What should we do?
  if (my_thread_id == *g_entered_thread_id_pointer) {
    // It looks the current thread is reentering the signal handler.
    // Something must be going wrong (maybe we are reentering by another
    // type of signal?).  Kill ourself by the default signal handler.
    InvokeDefaultSignalHandler(signal_number);
  }
  // Another thread is dumping stuff.  Let's wait until that thread
  // finishes the job and kills the process.
  while (true) {
    using namespace std::chrono_literals;
    std::this_thread::sleep_for(1s);
  }
}
```

上面这个while(true)会导致进程hang死，但是后面新的[pr](https://github.com/google/glog/commit/cafba580ec4faeeb8e61a97f91819fd0db3205a0#diff-59f5855888159e5e04a6bb3d6e2563e5f55fbbe29b081b8cc3fad424a4978c13)里已经换call_once改造过这里了，理论新版的glog就没这个问题

kernel里整体的代码调用链有点长，我本地调试用的6.12.4的内核版本

```c
/**
 *  sys_kill - send a signal to a process
 *  @pid: the PID of the process
 *  @sig: signal to be sent
 */
SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
{
	struct kernel_siginfo info;

	prepare_kill_siginfo(sig, &info, PIDTYPE_TGID);

	return kill_something_info(sig, &info, pid);
}
```

之后进入 kill_something_info, 看为什么kill执行忽略掉了

```c
/*
 * kill_something_info() interprets pid in interesting ways just like kill(2).
 *
 * POSIX specifies that kill(-1,sig) is unspecified, but what we have
 * is probably wrong.  Should make it like BSD or SYSV.
 */

static int kill_something_info(int sig, struct kernel_siginfo *info, pid_t pid)
{
	int ret;

	if (pid > 0)
		return kill_proc_info(sig, info, pid);

	/* -INT_MIN is undefined.  Exclude this case to avoid a UBSAN warning */
	if (pid == INT_MIN)
		return -ESRCH;

	read_lock(&tasklist_lock);
	if (pid != -1) {
		ret = __kill_pgrp_info(sig, info,
				pid ? find_vpid(-pid) : task_pgrp(current));
	} else {
		int retval = 0, count = 0;
		struct task_struct * p;

		for_each_process(p) {
			if (task_pid_vnr(p) > 1 &&
					!same_thread_group(p, current)) {
				int err = group_send_sig_info(sig, info, p,
							      PIDTYPE_MAX);
				++count;
				if (err != -EPERM)
					retval = err;
			}
		}
		ret = count ? retval : -ESRCH;
	}
	read_unlock(&tasklist_lock);

	return ret;
}

已知pid != -1, 只需要专注看__kill_pgrp_info->group_send_sig_info

```c
/*
 * __kill_pgrp_info() sends a signal to a process group: this is what the tty
 * control characters do (^C, ^Z etc)
 * - the caller must hold at least a readlock on tasklist_lock
 */
int __kill_pgrp_info(int sig, struct kernel_siginfo *info, struct pid *pgrp)
{
	struct task_struct *p = NULL;
	int ret = -ESRCH;

	do_each_pid_task(pgrp, PIDTYPE_PGID, p) {
		int err = group_send_sig_info(sig, info, p, PIDTYPE_PGID);
		/*
		 * If group_send_sig_info() succeeds at least once ret
		 * becomes 0 and after that the code below has no effect.
		 * Otherwise we return the last err or -ESRCH if this
		 * process group is empty.
		 */
		if (ret)
			ret = err;
	} while_each_pid_task(pgrp, PIDTYPE_PGID, p);

	return ret;
}
```

这里看起来是对一个group做batch操作，继续看 group_send_sig_info

```c
/*
 * send signal info to all the members of a thread group or to the
 * individual thread if type == PIDTYPE_PID.
 */
int group_send_sig_info(int sig, struct kernel_siginfo *info,
			struct task_struct *p, enum pid_type type)
{
	int ret;

	rcu_read_lock();
	ret = check_kill_permission(sig, info, p);
	rcu_read_unlock();

	if (!ret && sig)
		ret = do_send_sig_info(sig, info, p, type);

	return ret;
}

int do_send_sig_info(int sig, struct kernel_siginfo *info, struct task_struct *p,
			enum pid_type type)
{
	unsigned long flags;
	int ret = -ESRCH;

	if (lock_task_sighand(p, &flags)) {
		ret = send_signal_locked(sig, info, p, type);
		unlock_task_sighand(p, &flags);
	}

	return ret;
}

int send_signal_locked(int sig, struct kernel_siginfo *info,
		       struct task_struct *t, enum pid_type type)
{
	/* Should SIGKILL or SIGSTOP be received by a pid namespace init? */
	bool force = false;

	if (info == SEND_SIG_NOINFO) {
		/* Force if sent from an ancestor pid namespace */
		force = !task_pid_nr_ns(current, task_active_pid_ns(t));
	} else if (info == SEND_SIG_PRIV) {
		/* Don't ignore kernel generated signals */
		force = true;
	} else if (has_si_pid_and_uid(info)) {
		/* SIGKILL and SIGSTOP is special or has ids */
		struct user_namespace *t_user_ns;

		rcu_read_lock();
		t_user_ns = task_cred_xxx(t, user_ns);
		if (current_user_ns() != t_user_ns) {
			kuid_t uid = make_kuid(current_user_ns(), info->si_uid);
			info->si_uid = from_kuid_munged(t_user_ns, uid);
		}
		rcu_read_unlock();

		/* A kernel generated signal? */
		force = (info->si_code == SI_KERNEL);

		/* From an ancestor pid namespace? */
		if (!task_pid_nr_ns(current, task_active_pid_ns(t))) {
			info->si_pid = 0;
			force = true;
		}
	}
	return __send_signal_locked(sig, info, t, type, force);
}

static int __send_signal_locked(int sig, struct kernel_siginfo *info,
				struct task_struct *t, enum pid_type type, bool force)
{
	struct sigpending *pending;
	struct sigqueue *q;
	int override_rlimit;
	int ret = 0, result;

	lockdep_assert_held(&t->sighand->siglock);

	result = TRACE_SIGNAL_IGNORED;
	if (!prepare_signal(sig, t, force))
		goto ret;

	pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending : &t->pending;
	/*
	 * Short-circuit ignored signals and support queuing
	 * exactly one non-rt signal, so that we can get more
	 * detailed information about the cause of the signal.
	 */
	result = TRACE_SIGNAL_ALREADY_PENDING;
	if (legacy_queue(pending, sig))
		goto ret;

	result = TRACE_SIGNAL_DELIVERED;
	/*
	 * Skip useless siginfo allocation for SIGKILL and kernel threads.
	 */
	if ((sig == SIGKILL) || (t->flags & PF_KTHREAD))
		goto out_set;

	/*
	 * Real-time signals must be queued if sent by sigqueue, or
	 * some other real-time mechanism.  It is implementation
	 * defined whether kill() does so.  We attempt to do so, on
	 * the principle of least surprise, but since kill is not
	 * allowed to fail with EAGAIN when low on memory we just
	 * make sure at least one signal gets delivered and don't
	 * pass on the info struct.
	 */
	if (sig < SIGRTMIN)
		override_rlimit = (is_si_special(info) || info->si_code >= 0);
	else
		override_rlimit = 0;

	q = __sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit, 0);

	if (q) {
		list_add_tail(&q->list, &pending->list);
		switch ((unsigned long) info) {
		case (unsigned long) SEND_SIG_NOINFO:
			clear_siginfo(&q->info);
			q->info.si_signo = sig;
			q->info.si_errno = 0;
			q->info.si_code = SI_USER;
			q->info.si_pid = task_tgid_nr_ns(current,
							task_active_pid_ns(t));
			rcu_read_lock();
			q->info.si_uid =
				from_kuid_munged(task_cred_xxx(t, user_ns),
						 current_uid());
			rcu_read_unlock();
			break;
		case (unsigned long) SEND_SIG_PRIV:
			clear_siginfo(&q->info);
			q->info.si_signo = sig;
			q->info.si_errno = 0;
			q->info.si_code = SI_KERNEL;
			q->info.si_pid = 0;
			q->info.si_uid = 0;
			break;
		default:
			copy_siginfo(&q->info, info);
			break;
		}
	} else if (!is_si_special(info) &&
		   sig >= SIGRTMIN && info->si_code != SI_USER) {
		/*
		 * Queue overflow, abort.  We may abort if the
		 * signal was rt and sent by user using something
		 * other than kill().
		 */
		result = TRACE_SIGNAL_OVERFLOW_FAIL;
		ret = -EAGAIN;
		goto ret;
	} else {
		/*
		 * This is a silent loss of information.  We still
		 * send the signal, but the *info bits are lost.
		 */
		result = TRACE_SIGNAL_LOSE_INFO;
	}

out_set:
	signalfd_notify(t, sig);
	sigaddset(&pending->signal, sig);

	/* Let multiprocess signals appear after on-going forks */
	if (type > PIDTYPE_TGID) {
		struct multiprocess_signals *delayed;
		hlist_for_each_entry(delayed, &t->signal->multiprocess, node) {
			sigset_t *signal = &delayed->signal;
			/* Can't queue both a stop and a continue signal */
			if (sig == SIGCONT)
				sigdelsetmask(signal, SIG_KERNEL_STOP_MASK);
			else if (sig_kernel_stop(sig))
				sigdelset(signal, SIGCONT);
			sigaddset(signal, sig);
		}
	}

	complete_signal(sig, t, type);
ret:
	trace_signal_generate(sig, info, t, type != PIDTYPE_PID, result);
	return ret;
}
```

在上述代码中，看到`TRACE_SIGNAL_IGNORED`，随后进到了`prepare_signal`函数详细看下

```c
/*
 * Handle magic process-wide effects of stop/continue signals. Unlike
 * the signal actions, these happen immediately at signal-generation
 * time regardless of blocking, ignoring, or handling.  This does the
 * actual continuing for SIGCONT, but not the actual stopping for stop
 * signals. The process stop is done as a signal action for SIG_DFL.
 *
 * Returns true if the signal should be actually delivered, otherwise
 * it should be dropped.
 */
static bool prepare_signal(int sig, struct task_struct *p, bool force)
{
	struct signal_struct *signal = p->signal;
	struct task_struct *t;
	sigset_t flush;

	if (signal->flags & SIGNAL_GROUP_EXIT) {
		if (signal->core_state)
			return sig == SIGKILL;
		/*
		 * The process is in the middle of dying, drop the signal.
		 */
		return false;
	} else if (sig_kernel_stop(sig)) {
		/*
		 * This is a stop signal.  Remove SIGCONT from all queues.
		 */
		siginitset(&flush, sigmask(SIGCONT));
		flush_sigqueue_mask(&flush, &signal->shared_pending);
		for_each_thread(p, t)
			flush_sigqueue_mask(&flush, &t->pending);
	} else if (sig == SIGCONT) {
		unsigned int why;
		/*
		 * Remove all stop signals from all queues, wake all threads.
		 */
		siginitset(&flush, SIG_KERNEL_STOP_MASK);
		flush_sigqueue_mask(&flush, &signal->shared_pending);
		for_each_thread(p, t) {
			flush_sigqueue_mask(&flush, &t->pending);
			task_clear_jobctl_pending(t, JOBCTL_STOP_PENDING);
			if (likely(!(t->ptrace & PT_SEIZED))) {
				t->jobctl &= ~JOBCTL_STOPPED;
				wake_up_state(t, __TASK_STOPPED);
			} else
				ptrace_trap_notify(t);
		}

		/*
		 * Notify the parent with CLD_CONTINUED if we were stopped.
		 *
		 * If we were in the middle of a group stop, we pretend it
		 * was already finished, and then continued. Since SIGCHLD
		 * doesn't queue we report only CLD_STOPPED, as if the next
		 * CLD_CONTINUED was dropped.
		 */
		why = 0;
		if (signal->flags & SIGNAL_STOP_STOPPED)
			why |= SIGNAL_CLD_CONTINUED;
		else if (signal->group_stop_count)
			why |= SIGNAL_CLD_STOPPED;

		if (why) {
			/*
			 * The first thread which returns from do_signal_stop()
			 * will take ->siglock, notice SIGNAL_CLD_MASK, and
			 * notify its parent. See get_signal().
			 */
			signal_set_stop_flags(signal, why | SIGNAL_STOP_CONTINUED);
			signal->group_stop_count = 0;
			signal->group_exit_code = 0;
		}
	}

	return !sig_ignored(p, sig, force);
}
```

随后进入sig_ignored函数->sig_task_ignored，具体瞅瞅为啥可以ignore

```c
static bool sig_ignored(struct task_struct *t, int sig, bool force)
{
	/*
	 * Blocked signals are never ignored, since the
	 * signal handler may change by the time it is
	 * unblocked.
	 */
	if (sigismember(&t->blocked, sig) || sigismember(&t->real_blocked, sig))
		return false;

	/*
	 * Tracers may want to know about even ignored signal unless it
	 * is SIGKILL which can't be reported anyway but can be ignored
	 * by SIGNAL_UNKILLABLE task.
	 */
	if (t->ptrace && sig != SIGKILL)
		return false;

	return sig_task_ignored(t, sig, force);
}

static bool sig_task_ignored(struct task_struct *t, int sig, bool force)
{
	void __user *handler;

	handler = sig_handler(t, sig);

	/* SIGKILL and SIGSTOP may not be sent to the global init */
	if (unlikely(is_global_init(t) && sig_kernel_only(sig)))
		return true;

	if (unlikely(t->signal->flags & SIGNAL_UNKILLABLE) &&
	    handler == SIG_DFL && !(force && sig_kernel_only(sig)))
		return true;

	/* Only allow kernel generated signals to this kthread */
	if (unlikely((t->flags & PF_KTHREAD) &&
		     (handler == SIG_KTHREAD_KERNEL) && !force))
		return true;

	return sig_handler_ignored(handler, sig);
}
```

最终看到，flags如果是SIGNAL_UNKILLABLE，当前函数会返回true，信号会被忽略掉

然后去看第一个SIGNAL_UNKILLABLE

```c
#define SIGNAL_UNKILLABLE	0x00000040 /* for init: ignore fatal signals */
```

后面感兴趣的时候就看了下这个flags是怎么被设置成SIGNAL_UNKILLABLE的，主要调用代码太多，就放一下具体函数的call流程

```c
// 从start_kernel开始
void __init start_kernel(void)
{
    ...
    rest_init();
    ...
}

static void __init rest_init(void)
{
    ...
    pid = user_mode_thread(kernel_init, NULL, CLONE_FS);
    ...
}

/*
* Create a user mode thread.
*/
pid_t user_mode_thread(int (*fn)(void *), void *arg, unsigned long flags)
{
    struct kernel_clone_args args = {
        .flags = ((lower_32_bits(flags) | CLONE_VM |
        CLONE_UNTRACED) & ~CSIGNAL),
        .exit_signal = (lower_32_bits(flags) & CSIGNAL),
        .fn = fn,
        .fn_arg = arg,
    };
    return kernel_clone(&args);
}
/*
 *  Ok, this is the main fork-routine.
 *
 * It copies the process, and if successful kick-starts
 * it and waits for it to finish using the VM if required.
 *
 * args->exit_signal is expected to be checked for sanity by the caller.
 */
pid_t kernel_clone(struct kernel_clone_args *args) {
  struct task_struct *p;
  // ...
  p = copy_process(NULL, trace, NUMA_NO_NODE, args);
  // ...
}
/*
 * This creates a new process as a copy of the old one,
 * but does not actually start it yet.
 *
 * It copies the registers, and all the appropriate
 * parts of the process environment (as per the clone
 * flags). The actual kick-off is left to the caller.
 */
__latent_entropy struct task_struct *copy_process(
                    struct pid *pid,
                    int trace,
                    int node,
                    struct kernel_clone_args *args)
{
  // ...
    if (is_child_reaper(pid)) {
        ns_of_pid(pid)->child_reaper = p;
        p->signal->flags |= SIGNAL_UNKILLABLE;
    }
  //...
}


/*
 * is_child_reaper returns true if the pid is the init process
 * of the current namespace. As this one could be checked before
 * pid_ns->child_reaper is assigned in copy_process, we check
 * with the pid number.
 */
static inline bool is_child_reaper(struct pid *pid)
{
    return pid->numbers[pid->level].nr == 1;
}
```

### 容器的pid namespace

TBD

### 容器pid 1的进程

TBD


### REF

1. [理解Docker容器的进程管理(PID1进程(容器内kill命令无法杀死)、进程信号处理、僵尸进程)](https://blog.csdn.net/m0_45406092/article/details/119007685)
