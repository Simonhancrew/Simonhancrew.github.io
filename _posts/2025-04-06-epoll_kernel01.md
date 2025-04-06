---
title: epoll的内核实现01
date: 2025-04-06 12:31:00 +0800
categories: [Blogging, kernel]
tags: [writing]
---

新一点的内核，我直接看的6.17.3的kernel代码，这部分的实现都在`/fs/evetpoll.c`

其实只想知道这玩意是不是线程安全的，但单看注释其实就大概知道这玩意是线程安全的

```c
/*
 * LOCKING:
 * There are three level of locking required by epoll :
 *
 * 1) epnested_mutex (mutex)
 * 2) ep->mtx (mutex)
 * 3) ep->lock (rwlock)
 *
 * The acquire order is the one listed above, from 1 to 3.
 * We need a rwlock (ep->lock) because we manipulate objects
 * from inside the poll callback, that might be triggered from
 * a wake_up() that in turn might be called from IRQ context.
 * So we can't sleep inside the poll callback and hence we need
 * a spinlock. During the event transfer loop (from kernel to
 * user space) we could end up sleeping due a copy_to_user(), so
 * we need a lock that will allow us to sleep. This lock is a
 * mutex (ep->mtx). It is acquired during the event transfer loop,
 * during epoll_ctl(EPOLL_CTL_DEL) and during eventpoll_release_file().
 * The epnested_mutex is acquired when inserting an epoll fd onto another
 * epoll fd. We do this so that we walk the epoll tree and ensure that this
 * insertion does not create a cycle of epoll file descriptors, which
 * could lead to deadlock. We need a global mutex to prevent two
 * simultaneous inserts (A into B and B into A) from racing and
 * constructing a cycle without either insert observing that it is
 * going to.
 * It is necessary to acquire multiple "ep->mtx"es at once in the
 * case when one epoll fd is added to another. In this case, we
 * always acquire the locks in the order of nesting (i.e. after
 * epoll_ctl(e1, EPOLL_CTL_ADD, e2), e1->mtx will always be acquired
 * before e2->mtx). Since we disallow cycles of epoll file
 * descriptors, this ensures that the mutexes are well-ordered. In
 * order to communicate this nesting to lockdep, when walking a tree
 * of epoll file descriptors, we use the current recursion depth as
 * the lockdep subkey.
 * It is possible to drop the "ep->mtx" and to use the global
 * mutex "epnested_mutex" (together with "ep->lock") to have it working,
 * but having "ep->mtx" will make the interface more scalable.
 * Events that require holding "epnested_mutex" are very rare, while for
 * normal operations the epoll private "ep->mtx" will guarantee
 * a better scalability.
 */
```

这个注释单看有三级锁机制, 简单翻译总结下

### 三级锁结构

1. **epnested_mutex (全局互斥锁)**
   - 用途：防止多个epoll fd互相插入形成循环依赖
   - 场景：当把一个epoll fd插入到另一个epoll fd时使用

2. **ep->mtx (每个epoll实例的互斥锁)**
   - 用途：保护事件传输循环、EPOLL_CTL_DEL操作和文件释放
   - 特点：允许睡眠，用于可能阻塞的操作(如copy_to_user)

3. **ep->lock (每个epoll实例的读写锁)**
   - 用途：保护从poll回调中操作的对象
   - 特点：自旋锁，用于IRQ上下文(不能睡眠)

加锁顺序其实是严格要求的，在`epoll_ctl`里就可以看到

## epoll_ctl的实现

```c
/*
 * The following function implements the controller interface for
 * the eventpoll file that enables the insertion/removal/change of
 * file descriptors inside the interest set.
 */
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
		struct epoll_event __user *, event)
{
	struct epoll_event epds;

	if (ep_op_has_event(op) &&
	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
		return -EFAULT;

	return do_epoll_ctl(epfd, op, fd, &epds, false);
}
```

具体的内核实现

```c
static inline int epoll_mutex_lock(struct mutex *mutex, int depth,
				   bool nonblock)
{
	if (!nonblock) {
		mutex_lock_nested(mutex, depth);
		return 0;
	}
	if (mutex_trylock(mutex))
		return 0;
	return -EAGAIN;
}

int do_epoll_ctl(int epfd, int op, int fd, struct epoll_event *epds,
		 bool nonblock)
{
	int error;
	int full_check = 0;
	struct eventpoll *ep;
	struct epitem *epi;
	struct eventpoll *tep = NULL;

	CLASS(fd, f)(epfd);
	if (fd_empty(f))
		return -EBADF;

	/* Get the "struct file *" for the target file */
	CLASS(fd, tf)(fd);
	if (fd_empty(tf))
		return -EBADF;

	/* The target file descriptor must support poll */
	if (!file_can_poll(fd_file(tf)))
		return -EPERM;

  /* 检查EPOLLWAKEUP标志 */  
	/* Check if EPOLLWAKEUP is allowed */
	if (ep_op_has_event(op))
		ep_take_care_of_epollwakeup(epds);

	/*
	 * We have to check that the file structure underneath the file descriptor
	 * the user passed to us _is_ an eventpoll file. And also we do not permit
	 * adding an epoll file descriptor inside itself.
	 */
	error = -EINVAL;
	if (fd_file(f) == fd_file(tf) || !is_file_epoll(fd_file(f)))
		goto error_tgt_fput;

	/*
	 * epoll adds to the wakeup queue at EPOLL_CTL_ADD time only,
	 * so EPOLLEXCLUSIVE is not allowed for a EPOLL_CTL_MOD operation.
	 * Also, we do not currently supported nested exclusive wakeups.
	 */
	if (ep_op_has_event(op) && (epds->events & EPOLLEXCLUSIVE)) {
		if (op == EPOLL_CTL_MOD)  // MOD操作不允许EPOLLEXCLUSIVE
			goto error_tgt_fput;
		if (op == EPOLL_CTL_ADD && (is_file_epoll(fd_file(tf)) ||
				(epds->events & ~EPOLLEXCLUSIVE_OK_BITS)))
			goto error_tgt_fput;
	}

	/*
	 * At this point it is safe to assume that the "private_data" contains
	 * our own data structure.
	 */
	ep = fd_file(f)->private_data;

	/*
	 * When we insert an epoll file descriptor inside another epoll file
	 * descriptor, there is the chance of creating closed loops, which are
	 * better be handled here, than in more critical paths. While we are
	 * checking for loops we also determine the list of files reachable
	 * and hang them on the tfile_check_list, so we can check that we
	 * haven't created too many possible wakeup paths.
	 *
	 * We do not need to take the global 'epumutex' on EPOLL_CTL_ADD when
	 * the epoll file descriptor is attaching directly to a wakeup source,
	 * unless the epoll file descriptor is nested. The purpose of taking the
	 * 'epnested_mutex' on add is to prevent complex toplogies such as loops and
	 * deep wakeup paths from forming in parallel through multiple
	 * EPOLL_CTL_ADD operations.
	 */
	error = epoll_mutex_lock(&ep->mtx, 0, nonblock);
	if (error)
		goto error_tgt_fput;
  // 针对add操作，需要全局锁和完整检查
	if (op == EPOLL_CTL_ADD) {
		if (READ_ONCE(fd_file(f)->f_ep) || ep->gen == loop_check_gen ||
		    is_file_epoll(fd_file(tf))) {
			mutex_unlock(&ep->mtx);
			error = epoll_mutex_lock(&epnested_mutex, 0, nonblock);
			if (error)
				goto error_tgt_fput;
			loop_check_gen++;
			full_check = 1;
			if (is_file_epoll(fd_file(tf))) {
				tep = fd_file(tf)->private_data;
				error = -ELOOP;
				if (ep_loop_check(ep, tep) != 0) // 嵌套检查
					goto error_tgt_fput;
			}
			error = epoll_mutex_lock(&ep->mtx, 0, nonblock);
			if (error)
				goto error_tgt_fput;
		}
	}

	/*
	 * Try to lookup the file inside our RB tree. Since we grabbed "mtx"
	 * above, we can be sure to be able to use the item looked up by
	 * ep_find() till we release the mutex.
	 */
	epi = ep_find(ep, fd_file(tf), fd);

	error = -EINVAL;
	switch (op) {
	case EPOLL_CTL_ADD:
		if (!epi) {
			epds->events |= EPOLLERR | EPOLLHUP;
			error = ep_insert(ep, epds, fd_file(tf), fd, full_check);
		} else
			error = -EEXIST;
		break;
	case EPOLL_CTL_DEL:
		if (epi) {
			/*
			 * The eventpoll itself is still alive: the refcount
			 * can't go to zero here.
			 */
			ep_remove_safe(ep, epi);
			error = 0;
		} else {
			error = -ENOENT;
		}
		break;
	case EPOLL_CTL_MOD:
		if (epi) {
			if (!(epi->event.events & EPOLLEXCLUSIVE)) {
				epds->events |= EPOLLERR | EPOLLHUP;
				error = ep_modify(ep, epi, epds);
			}
		} else
			error = -ENOENT;
		break;
	}
	mutex_unlock(&ep->mtx);

error_tgt_fput:
	if (full_check) {
		clear_tfile_check_list();
		loop_check_gen++;
		mutex_unlock(&epnested_mutex);
	}
	return error;
}
```

这里面其实是有锁的，在`error = epoll_mutex_lock(&ep->mtx, 0, nonblock);`这里加了ep->mtx的锁.

普通操作只使用实例锁，嵌套epoll操作需要全局锁。

这里epoll的结构体其实如下

```c
/*
 * This structure is stored inside the "private_data" member of the file
 * structure and represents the main data structure for the eventpoll
 * interface.
 */
struct eventpoll {
	/*
	 * This mutex is used to ensure that files are not removed
	 * while epoll is using them. This is held during the event
	 * collection loop, the file cleanup path, the epoll file exit
	 * code and the ctl operations.
	 */
	struct mutex mtx;

	/* Wait queue used by sys_epoll_wait() */
	wait_queue_head_t wq;

	/* Wait queue used by file->poll() */
	wait_queue_head_t poll_wait;

	/* List of ready file descriptors */
	struct list_head rdllist;

	/* Lock which protects rdllist and ovflist */
	rwlock_t lock;

	/* RB tree root used to store monitored fd structs */
	struct rb_root_cached rbr;

	/*
	 * This is a single linked list that chains all the "struct epitem" that
	 * happened while transferring ready events to userspace w/out
	 * holding ->lock.
	 */
	struct epitem *ovflist;

	/* wakeup_source used when ep_send_events or __ep_eventpoll_poll is running */
	struct wakeup_source *ws;

	/* The user that created the eventpoll descriptor */
	struct user_struct *user;

	struct file *file;

	/* used to optimize loop detection check */
	u64 gen;
	struct hlist_head refs;

	/*
	 * usage count, used together with epitem->dying to
	 * orchestrate the disposal of this struct
	 */
	refcount_t refcount;

#ifdef CONFIG_NET_RX_BUSY_POLL
	/* used to track busy poll napi_id */
	unsigned int napi_id;
	/* busy poll timeout */
	u32 busy_poll_usecs;
	/* busy poll packet budget */
	u16 busy_poll_budget;
	bool prefer_busy_poll;
#endif

#ifdef CONFIG_DEBUG_LOCK_ALLOC
	/* tracks wakeup nests for lockdep validation */
	u8 nests;
#endif
};
```
### EPOLLEXCLUSIVE

另外这边其实看到还有EPOLLEXCLUSIVE，这玩意主要处理惊群的时候用的

EPOLLEXCLUSIVE 标志确保当事件发生时，只有一个等待的 epoll 实例会被唤醒，而不是唤醒所有监听者。

当多个进程/线程通过 epoll 同时监听同一个文件描述符（如 socket）时，如果该文件描述符上有事件发生，内核会唤醒所有监听者，但最终只有一个进程/线程能成功处理该事件，其他被唤醒的进程/线程会白白消耗CPU资源，这种现象称为"惊群"。

这个其实在上面的代码里有写，这玩意只能在add的时候加，不能mod的时候弄

```c
/*
  * epoll adds to the wakeup queue at EPOLL_CTL_ADD time only,
  * so EPOLLEXCLUSIVE is not allowed for a EPOLL_CTL_MOD operation.
  * Also, we do not currently supported nested exclusive wakeups.
  */
if (ep_op_has_event(op) && (epds->events & EPOLLEXCLUSIVE)) {
  if (op == EPOLL_CTL_MOD)
    goto error_tgt_fput;
  if (op == EPOLL_CTL_ADD && (is_file_epoll(fd_file(tf)) ||
      (epds->events & ~EPOLLEXCLUSIVE_OK_BITS)))
    goto error_tgt_fput;
}
```

总体没看懂，下次主要看下`ep_poll_callback`相关的代码
