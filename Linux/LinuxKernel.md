# Process Management

### fork
![Linux fork](../Images/linux-fork.jpg)

### exec
```C++
do_execve(getname(filename), argv, envp);
    do_execveat_common
        exec_binprm
            search_binary_handler // fs/exec.c
                load_elf_binary // fs/binfmt_elf.c
                    setup_new_exec(); // set mmap_base
                    setup_arg_pages();
                    elf_map(); // map the code in elf file to memory
                    set_brk(); // setup heap area
                    load_elf_interp(); // load dependent *.so
                    start_thread(regs, elf_entry, bprm->p); // arch/x86/kernel/process_32.c
```

### pthread
```C++
// Linux pthread implementation
__pthread_create_2_1 // binfmt_elf.c
    ALLOCATE_STACK (iattr, &pd); // allocatestack.c
        /* Adjust the stack size for alignment. */
        /* call mmap to alloc thread statck in process heap*/
        /* set protection of this thread stack memory */
        /* populate member: stackblock、stackblock_size、guardsize、specific */
        /* And add to the list of stacks in use. */
    // start thread
    create_thread (pd, iattr, stackaddr, stacksize); // sysdeps/pthread/createthread.c
    do_clone (pd, attr, clone_flags, start_thread/*pthread func*/, stackaddr, stacksize, 1);
        ARCH_CLONE (start_thread, stackaddr, stacksize,, clone_flags, pd, &pd->tid, TLS_VALUE, &pd->tid);
            _do_fork();
    start_thread // pthread_creat.c
        THREAD_SETMEM (pd, result, pd->start_routine (pd->arg));
        // pd->result = pd->start_routine(pd->arg);
    __free_tcb
        __deallocate_stack
            queue_stack
```

### schedule
##### Voluntary Schedule
```c++
schedule(void)
    __schedule(false), // kernel/sched/core.c
        pick_next_task(rq, prev, &rf);
        context_switch(rq, prev, next, &rf);
            switch_mm_irqs_off(prev->active_mm, next->mm, next);
            switch_to(prev, next, prev);
                __switch_to_asm(); // switch registers, but not EIP [arch/x86/entry/entry_64.S]
                    __switch_to(); // switch stack [arch/x86/kernel/process_32.c]
                        this_cpu_write(current_task, next_p); //
            barrier();
            return finish_task_switch(prev);
```


##### Involuntary Shcedule(preempty)
```C++
// 1. mark TIF_NEED_RESCHED
// no time slice
scheduler_tick(); // kernel/sched/core.c
    task_tick_fair(rq, curr, 0);
        entity_tick(cfs_rq, se, queued);
            check_preempt_tick(cfs_rq, curr);
                resched_curr(rq_of(cfs_rq));
                    set_tsk_need_resched();

// wake up
try_to_wake_up(); // kernel/sched/core.c
    ttwu_queue
        ttwu_do_activate
            ttwu_do_wakeup
                check_preempt_curr
                    resched_curr
                        --->

// 2. real time shceudle
// real user space preempty time: 1. return from system call
do_syscall_64();
    syscall_return_slowpath();
        prepare_exit_to_usermode();
            exit_to_usermode_loop(struct pt_regs *regs, u32 cached_flags) {
                while (true) {
                    if (cached_flags & _TIF_NEED_RESCHED)
                        schedule();

                    if (cached_flags & _TIF_SIGPENDING) {
                        do_signal(regs) {
                            struct ksignal ksig;
                            if (get_signal(&ksig)) {
                                handle_signal(&ksig, regs) {
                                    bool stepping, failed;
                                    if (syscall_get_nr(current, regs) >= 0) {
                                        switch (syscall_get_error(current, regs)) {
                                        case -ERESTART_RESTARTBLOCK:
                                        case -ERESTARTNOHAND:
                                            regs->ax = -EINTR;
                                            break;
                                        case -ERESTARTSYS:
                                            if (!(ksig->ka.sa.sa_flags & SA_RESTART)) {
                                                regs->ax = -EINTR;
                                                break;
                                            }
                                        case -ERESTARTNOINTR:
                                            regs->ax = regs->orig_ax;
                                            regs->ip -= 2;
                                            break;
                                        }
                                    }
                                    failed = (setup_rt_frame(ksig, regs) < 0) {
                                        static int __setup_rt_frame(int sig, struct ksignal *ksig,
                                                sigset_t *set, struct pt_regs *regs)
                                        {
                                            struct rt_sigframe __user *frame;
                                            void __user *fp = NULL;
                                            int err = 0;

                                            frame = get_sigframe(&ksig->ka, regs, sizeof(struct rt_sigframe), &fp);
                                            put_user_try {
                                                if (ksig->ka.sa.sa_flags & SA_RESTORER) {
                                                    put_user_ex(ksig->ka.sa.sa_restorer, &frame->pretcode);
                                                }
                                            } put_user_catch(err);

                                            err |= setup_sigcontext(&frame->uc.uc_mcontext, fp, regs, set->sig[0]);
                                            err |= __copy_to_user(&frame->uc.uc_sigmask, set, sizeof(*set));

                                            regs->di = sig;
                                            regs->ax = 0;

                                            regs->si = (unsigned long)&frame->info;
                                            regs->dx = (unsigned long)&frame->uc;
                                            regs->ip = (unsigned long) ksig->ka.sa.sa_handler;

                                            regs->sp = (unsigned long)frame;
                                            regs->cs = __USER_CS;
                                            return 0;
                                        }
                                    }
                                    signal_setup_done(failed, ksig, stepping);
                                }
                                return;
                            }

                            if (syscall_get_nr(current, regs) >= 0) {
                                switch (syscall_get_error(current, regs)) {
                                case -ERESTARTNOHAND:
                                case -ERESTARTSYS:
                                case -ERESTARTNOINTR:
                                    regs->ax = regs->orig_ax;
                                    regs->ip -= 2;
                                    break;
                                case -ERESTART_RESTARTBLOCK:
                                    regs->ax = get_nr_restart_syscall(regs);
                                    regs->ip -= 2;
                                    break;
                                }
                            }
                            restore_saved_sigmask();
                        }
                    }

                    if (!(cached_flags & EXIT_TO_USERMODE_LOOP_FLAGS))
                        break;
                }
            }


    // restore user space stack upon return from sighandler
    asmlinkage long sys_rt_sigreturn(void)
    {
        struct pt_regs *regs = current_pt_regs();
        struct rt_sigframe __user *frame;
        sigset_t set;
        unsigned long uc_flags;

        frame = (struct rt_sigframe __user *)(regs->sp - sizeof(long));
        if (__copy_from_user(&set, &frame->uc.uc_sigmask, sizeof(set)))
            goto badframe;
        if (__get_user(uc_flags, &frame->uc.uc_flags))
            goto badframe;

        set_current_blocked(&set);

        if (restore_sigcontext(regs, &frame->uc.uc_mcontext, uc_flags))
            goto badframe;
        return regs->ax;
    }

// real user space preempty time: 2. return from interrupt
do_IRQ
    retint_user //arch/x86/entry/entry_64.S
        prepare_exit_to_usermode
            exit_to_usermode_loop
                --->


// real kernel preempty time: 1. preempty_enble
preempt_enable
    preempt_count_dec_and_test
        preempt_schedule
            preempt_schedule_common
                __schedule
                    --->

// real kernel preempty time: 2. return from interrupt
do_IRQ
    retint_kernel
        prepare_exit_to_usermode
            --->

```

# Signal
```C++
struct task_struct {
    /* Signal handlers: */
    struct signal_struct		*signal;
    struct sighand_struct __rcu		*sighand;
    sigset_t			blocked;
    sigset_t			real_blocked;
    /* Restored if set_restore_sigmask() was used: */
    sigset_t			saved_sigmask;
    struct sigpending		pending;
    unsigned long			sas_ss_sp;
    size_t				sas_ss_size;
    unsigned int			sas_ss_flags;
};
/*
kill->kill_something_info->kill_pid_info->group_send_sig_info->do_send_sig_info
tkill->do_tkill->do_send_specific->do_send_sig_info
tgkill->do_tkill->do_send_specific->do_send_sig_info
rt_sigqueueinfo->do_rt_sigqueueinfo->kill_proc_info->kill_pid_info->group_send_sig_info->do_send_sig_info
*/


do_send_sig_info() {
    send_signal() {
        __send_signal(int sig, struct siginfo *info, struct task_struct *t, int group, int from_ancestor_ns) {
            struct sigpending *pending;
            struct sigqueue *q;
            int override_rlimit;
            int ret = 0, result;
            pending = group ? &t->signal->shared_pending : &t->pending;
            if (legacy_queue(pending, sig) {
                return (sig < SIGRTMIN) && sigismember(&pending->signal, sig);
            })
                goto ret;

            if (sig < SIGRTMIN)
                override_rlimit = (is_si_special(info) || info->si_code >= 0);
            else
                override_rlimit = 0;

            q = __sigqueue_alloc(sig, t, GFP_ATOMIC | __GFP_NOTRACK_FALSE_POSITIVE,
                override_rlimit);
            if (q) {
                list_add_tail(&q->list, &pending->list);
                switch ((unsigned long) info) {
                case (unsigned long) SEND_SIG_NOINFO:
                    q->info.si_signo = sig;
                    q->info.si_errno = 0;
                    q->info.si_code = SI_USER;
                    q->info.si_pid = task_tgid_nr_ns(current,
                            task_active_pid_ns(t));
                    q->info.si_uid = from_kuid_munged(current_user_ns(), current_uid());
                    break;
                case (unsigned long) SEND_SIG_PRIV:
                    q->info.si_signo = sig;
                    q->info.si_errno = 0;
                    q->info.si_code = SI_KERNEL;
                    q->info.si_pid = 0;
                    q->info.si_uid = 0;
                    break;
                default:
                    copy_siginfo(&q->info, info);
                    if (from_ancestor_ns)
                        q->info.si_pid = 0;
                    break;
                }
                userns_fixup_signal_uid(&q->info, t);

            }
            out_set:
            signalfd_notify(t, sig);
            sigaddset(&pending->signal, sig);
            complete_signal(sig, t, group) {
                struct signal_struct *signal = p->signal;
                struct task_struct *t;
                if (wants_signal(sig, p))
                    t = p;
                else if (!group || thread_group_empty(p))
                    return;
                else {
                    t = signal->curr_target;
                    while (!wants_signal(sig, t)) {
                        t = next_thread(t);
                        if (t == signal->curr_target)
                            return;
                    }
                    signal->curr_target = t;
                }
                signal_wake_up(t, sig == SIGKILL) {
                    set_tsk_thread_flag(t, TIF_SIGPENDING);
                    if (!wake_up_state(t, state | TASK_INTERRUPTIBLE))
                        kick_process(t);
                }
            }
            ret:
            return ret;
        }
    }
}

// signal real handing see <Modern Operating System.md> exit_to_usermode_loop
```
![Linux signal handling](../Images/linux-signal-handling.png)

# Memory Management
### malloc
```C++
// mm/mmap.c
SYSCALL_DEFINE1(brk, unsigned long, brk)
{
    unsigned long retval;
    unsigned long newbrk, oldbrk;
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *next;
    //
    newbrk = PAGE_ALIGN(brk);
    oldbrk = PAGE_ALIGN(mm->brk);
    if (oldbrk == newbrk)
        goto set_brk;

    /* Always allow shrinking brk. */
    if (brk <= mm->brk) {
        if (!do_munmap(mm, newbrk, oldbrk-newbrk, &uf))
            goto set_brk;
        goto out;
    }

    /* Check against existing mmap mappings. */
    next = find_vma(mm, oldbrk);
    if (next && newbrk + PAGE_SIZE > vm_start_gap(next))
        goto out;

    /* Ok, looks good - let it rip. */
    if (do_brk(oldbrk, newbrk-oldbrk, &uf) < 0)
        do_brk_flags();
            find_vma_links();
            vma_merge();
            kmem_cache_zalloc();
            INIT_LIST_HEAD(&vma->anon_vma_chain);
            vma_link(mm, vma, prev, rb_link, rb_parent); // insert mm_struct rb_mm;
        goto out;

set_brk:
  mm->brk = brk;
//
  return brk;
out:
  retval = mm->brk;
  return retval
```

### mmap
```C++
// Linux mmap implementation
ksys_mmap_pgoff(); // mm/mmap.c
    vm_mmap_pgoff();
        do_mmap_pgoff();
            do_mmap();
                get_unmapped_area() {
                    unsigned long (*get_area)(struct file *, unsigned long, unsigned long,
                        unsigned long, unsigned long);
                    get_area = current->mm->get_unmapped_area;   // 1. anonymous mapping
                    if (file) {
                        get_area = file->f_op->get_unmapped_area; // 2. file mapping
                    } else if (flags & MAP_SHARED) {
                        get_area = shmem_get_unmapped_area;      // 3. shmem mapping
                    }
                    addr = get_area(file, addr, len, pgoff, flags);
                }
                mmap_region() {
                    vma = vma_merge(mm, prev, addr, addr + len, vm_flags,
                            NULL, file, pgoff, NULL, NULL_VM_UFFD_CTX);
                    if (vma)
                        goto out;

                    vma = vm_area_alloc(mm);
                    if (!vma) {
                        error = -ENOMEM;
                        goto unacct_error;
                    }

                    vma->vm_start = addr;
                    vma->vm_end = addr + len;
                    vma->vm_flags = vm_flags;
                    vma->vm_page_prot = vm_get_page_prot(vm_flags);
                    vma->vm_pgoff = pgoff;

                    if (file) {
                        if (vm_flags & VM_DENYWRITE) {
                            error = deny_write_access(file);
                        }
                        if (vm_flags & VM_SHARED) {
                            error = mapping_map_writable(file->f_mapping);
                        }
                        vma->vm_file = get_file(file); // map file to vma
                        error = call_mmap(file, vma) {
                            file->f_op->mmap(file, vma) {
                                ext4_file_mmap() {
                                    vma->vm_ops = &ext4_file_vm_ops; // map file ops to vma ops
                                }
                            }
                        }
                    } else if (vm_flags & VM_SHARED) {
                        error = shmem_zero_setup(vma);
                    } else {
                        vma_set_anonymous(vma);
                    }

                    vma_link(mm, vma, prev, rb_link, rb_parent) {
                        __vma_link(mm, vma, prev, rb_link, rb_parent);
                        __vma_link_file(vma) {
                            vma_interval_tree_insert(vma, &mapping->i_mmap); // map file to vma
                        }
                    }
                }
```
```C++
// page fault exception
do_page_fault() {
    __do_page_fault();
        vmalloc_fault(address) { // fault in kernel

        }

        handle_mm_fault(vma, address, flags); // fault in usr space
            handle_pte_fault(&vmf);
                do_anonymous_page(vmf) { // 1. map physical memory
                    pte_alloc(); // alloc a page table item
                    alloc_zeroed_user_highpage_movable() {
                        alloc_pages_vma();
                        __alloc_pages_nodemask(); // alloc psyhic memory
                            --->
                    }
                    mk_pte(page, vma->vm_page_prot);
                    set_pte_at(vma->vm_mm, vmf->address, vmf->pte, entry);
                }

                do_fault(vmf) {          // 2. map to a file
                    __do_fault() {
                        vma->vm_ops->fault(vmf) {
                            ext4_filemap_fault() {
                                struct inode *inode = file_inode(vmf->vma->vm_file);
                                filemap_fault(vmf) {
                                    struct file *file = vmf->vma->vm_file;
                                    struct address_space *mapping = file->f_mapping;
                                    struct inode *inode = mapping->host;
                                    pgoff_t offset = vmf->pgoff;
                                    struct page *page; // page cache in physic of the file
                                    int ret = 0;

                                    page = find_get_page(mapping, offset);
                                    if (likely(page) && !(vmf->flags & FAULT_FLAG_TRIED)) {
                                        do_async_mmap_readahead(vmf->vma, ra, file, page, offset);
                                    } else if (!page) {
                                        goto no_cached_page;
                                    }

                                    vmf->page = page;
                                    return ret | VM_FAULT_LOCKED;

                                    no_cached_page:
                                    page_cache_read(file, offset, vmf->gfp_mask) {
                                        struct address_space *mapping = file->f_mapping;
                                        struct page *page;
                                        page = __page_cache_alloc(gfp_mask|__GFP_COLD);
                                        ret = add_to_page_cache_lru(page, mapping, offset, gfp_mask & GFP_KERNEL);
                                        ret = mapping->a_ops->readpage(file, page) {
                                            struct address_space *mapping = file->f_mapping;
                                            struct page *page;
                                            page = __page_cache_alloc(gfp_mask|__GFP_COLD);
                                            ret = add_to_page_cache_lru(page, mapping, offset, gfp_mask & GFP_KERNEL);
                                            ret = mapping->a_ops->readpage(file, page) {
                                                ext4_read_inline_page(inode, page) {
                                                    void *kaddr = kmap_atomic(page);
                                                    ret = ext4_read_inline_data(inode, kaddr, len, &iloc);
                                                    flush_dcache_page(page);
                                                    kunmap_atomic(kaddr);
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
                do_swap_page(vmf) {      // 3. map to swap
                    struct vm_area_struct *vma = vmf->vma;
                    struct page *page, *swapcache;
                    struct mem_cgroup *memcg;
                    swp_entry_t entry;
                    pte_t pte;
                    entry = pte_to_swp_entry(vmf->orig_pte);
                    page = lookup_swap_cache(entry);
                    if (!page) {
                        page = swapin_readahead(entry, GFP_HIGHUSER_MOVABLE, vma, vmf->address) {
                            int swap_readpage(struct page *page, bool do_poll) {
                                struct bio *bio;
                                int ret = 0;
                                struct swap_info_struct *sis = page_swap_info(page);
                                blk_qc_t qc;
                                struct block_device *bdev;
                                if (sis->flags & SWP_FILE) {
                                    struct file *swap_file = sis->swap_file;
                                    struct address_space *mapping = swap_file->f_mapping;
                                    ret = mapping->a_ops->readpage(swap_file, page);
                                    return ret;
                                }
                            }
                        }
                    }
                    swapcache = page;
                    pte = mk_pte(page, vma->vm_page_prot);
                    set_pte_at(vma->vm_mm, vmf->address, vmf->pte, pte);
                    vmf->orig_pte = pte;
                    swap_free(entry);
                }
}


static struct mm_struct *dup_mm(struct task_struct *tsk)
{
  struct mm_struct *mm, *oldmm = current->mm;
  mm = allocate_mm() {
      kmem_cache_alloc(mm_cachep, GFP_KERNEL);
  }
  memcpy(mm, oldmm, sizeof(*mm));
  mm_init(mm, tsk, mm->usr_ns) {
        mm_alloc_pgd(mm) {
            mm->pdg = pgd_alloc() { // arch/x86/mm/pgtable.c
                pgd_t *pgd = _pgd_alloc();
                pgd_ctor(mm, pgd) {
                    if (CONFIG_PGTABLE_LEVELS == 2 ||
                        (CONFIG_PGTABLE_LEVELS == 3 && SHARED_KERNEL_PMD) ||
                        CONFIG_PGTABLE_LEVELS >= 4) {
                        clone_pgd_range(pgd + KERNEL_PGD_BOUNDARY,
                            swapper_pg_dir + KERNEL_PGD_BOUNDARY,
                            KERNEL_PGD_PTRS);
                    }
                }
            }
        }
  }
  err = dup_mmap(mm, oldmm);
  return mm;
}
```

### Buddy System
```C++
typedef struct pglist_data {
  struct zone node_zones[MAX_NR_ZONES];
  struct zonelist node_zonelists[MAX_ZONELISTS];
  int nr_zones;
  struct page *node_mem_map;
  unsigned long node_start_pfn;
  unsigned long node_present_pages; /* total number of physical pages */
  unsigned long node_spanned_pages; /* total size of physical page range, including holes */
  int node_id;
} pg_data_t;


struct zone {
  struct pglist_data  *zone_pgdat;
  struct per_cpu_pageset __percpu *pageset;

  unsigned long    zone_start_pfn;

  unsigned long    managed_pages;
  unsigned long    spanned_pages;
  unsigned long    present_pages;

  const char    *name;
  struct free_area  free_area[MAX_ORDER];
  unsigned long    flags;
  spinlock_t    lock;
} ____cacheline_internodealigned_in_

// Linux buddy system
alloc_pages();
    alloc_pages_current(gfp_mask, order);
        struct mempolicy *pol = &default_policy;
        struct page *page;
        //
        page = __alloc_pages_nodemask(gfp, order, policy_node(gfp, pol, numa_node_id()), policy_nodemask(gfp, pol));
            get_page_from_freelist();
                //
                for_next_zone_zonelist_nodemask(zone, z, ac->zonelist, ac->high_zoneidx, ac->nodemask) {
                    struct page *page;
                    page = rmqueue(ac->preferred_zoneref->zone, zone, order, gfp_mask, alloc_flags, ac->migratetype);
                        __rmqueue();
                            rmqueue_smallest() {
                                /* Find a page of the appropriate size in the preferred list */
                                for (current_order = order; current_order < MAX_ORDER; ++current_order) {
                                    area = &(zone->free_area[current_order]);
                                    page = list_first_entry_or_null(&area->free_list[migratetype], struct page, lru);
                                    if (!page)
                                        continue;
                                    list_del(&page->lru);
                                    rmv_page_order(page);
                                    area->nr_free--;
                                    expand(zone, page, order, current_order, area, migratetype) {
                                        unsigned long size = 1 << high;
                                        while (high > low) {
                                            area--;
                                            high--;
                                            size >>= 1;
                                            list_add(&page[size].lru, &area->free_list[migratetype]);
                                            area->nr_free++;
                                            set_page_order(&page[size], high);
                                        }
                                    }

                                    set_pcppage_migratetype(page, migratetype); return page;
                                }
                                return NULL;
                            }
                }
        return page;
```
```C++
struct kmem_cache {
  struct kmem_cache_cpu __percpu *cpu_slab;
  /* Used for retriving partial slabs etc */
  unsigned long flags;
  unsigned long min_partial;
  int size;    /* The size of an object including meta data */
  int object_size;  /* The size of an object without meta data */
  int offset;    /* Free pointer offset. */
#ifdef CONFIG_SLUB_CPU_PARTIAL
  int cpu_partial;  /* Number of per cpu partial objects to keep around */
#endif
  struct kmem_cache_order_objects oo;
  /* Allocation and freeing of slabs */
  struct kmem_cache_order_objects max;
  struct kmem_cache_order_objects min;
  gfp_t allocflags;  /* gfp flags to use on each alloc */
  int refcount;    /* Refcount for slab cache destroy */
  void (*ctor)(void *);
  const char *name;  /* Name (only for display!) */
  struct list_head list;  /* List of slab caches */
  struct kmem_cache_node *node[MAX_NUMNODES];
};

// all chches will put into LIST_HEAD(slab_caches)

struct kmem_cache_cpu {
  void **freelist;  /* Pointer to next available object */
  unsigned long tid;  /* Globally unique transaction id */
  struct page *page;  /* The slab from which we are allocating */
#ifdef CONFIG_SLUB_CPU_PARTIAL
  struct page *partial;  /* Partially allocated frozen slabs */
#endif
};


struct kmem_cache_node {
  spinlock_t list_lock;
#ifdef CONFIG_SLUB
  unsigned long nr_partial;
  struct list_head partial;
#endif
};
```
![kmem_cache_cpu_node](../Images/kmem-cache-cpu-node.jpg)

### Slab/slub/slob System
```C++
// Linux slab/slub/slob system
static struct kmem_cache *task_struct_cachep;

static inline struct task_struct *alloc_task_struct_node(int node)
{
    return kmem_cache_alloc_node(task_struct_cachep, GFP_KERNEL, node);
}

static inline void free_task_struct(struct task_struct *tsk)
{
    kmem_cache_free(task_struct_cachep, tsk);
}


alloc_task_struct_node();
    kmem_cache_alloc_node(task_struct_cachep, GFP_KERNEL, node);
        slab_alloc_node();
            __slab_alloc() {
                // 1. try freelist again in case of cpu migration or IRQ
                freelist = get_freelist(s, page);

                // 2. replace cpu page with partial
                if (slub_percpu_partial(c)) {
                    page = c->page = slub_percpu_partial(c); // c->partial
                    slub_set_percpu_partial(c, page); // c->partical = page->next
                    stat(s, CPU_PARTIAL_ALLOC);
                    goto redo;
                }

                // 3. need alloc new slak objects
               freelist = new_slab_objects(struct kmem_cache *s, gfp_t flags,
                    int node, struct kmem_cache_cpu **pc)
                {
                    // 3.1. get partial from kmem_cache_node indexed by node
                    freelist = get_partial_node(struct kmem_cache *s, struct kmem_cache_node *n,
                            struct kmem_cache_cpu *c, gfp_t flags)
                    {
                        struct page *page, *page2;
                        void *object = NULL;
                        int available = 0;
                        int objects;
                        list_for_each_entry_safe(page, page2, &n->partial, lru) {
                            // get a page memory from kmem_cache_node, return its freelist
                            void *t = acquire_slab(s, n, page, object == NULL, &objects);
                            if (!t)
                            break;

                            available += objects;
                            if (!object) {
                                c->page = page;
                                stat(s, ALLOC_FROM_PARTIAL);
                                object = t;
                            } else {
                                put_cpu_partial(s, page, 0);
                                stat(s, CPU_PARTIAL_NODE);
                            }
                            if (!kmem_cache_has_cpu_partial(s)
                                || available > slub_cpu_partial(s) / 2)
                                break;
                        }
                        return object;
                    }
                    if (freelist)
                        return freelist;

                    // 3.2. no memory in kmem_cache_node, alloc new
                    page = new_slab(s, flags, node) {
                        page *allocate_slab(struct kmem_cache *s, gfp_t flags, int node)
                        {
                            struct page *page;
                            struct kmem_cache_order_objects oo = s->oo;
                            gfp_t alloc_gfp;
                            void *start, *p;
                            int idx, order;
                            bool shuffle;
                            flags &= gfp_allowed_mask;
                            page = alloc_slab_page(s, alloc_gfp, node, oo);
                            if (unlikely(!page)) {
                                oo = s->min;
                                alloc_gfp = flags;
                                page = alloc_slab_page(s, alloc_gfp, node, oo);
                                    __alloc_pages_node();
                                        __alloc_pages(gfp_mask, order, nid);
                                            __alloc_pages_nodemask();
                                                --->

                                if (unlikely(!page))
                                    goto out;
                                stat(s, ORDER_FALLBACK);
                            }
                            return page;
                        }
                    }
                    if (page) {
                        c = raw_cpu_ptr(s->cpu_slab);
                        if (c->page)
                            flush_slab(s, c);

                        freelist = page->freelist;
                        page->freelist = NULL;

                        stat(s, ALLOC_SLAB);
                        c->page = page;
                        *pc = c;
                    } else
                        freelist = NULL;
                    return freelis
                }
```

### kswapd
```C++
//1. actice page out when alloc
get_page_from_freelist();
    node_reclaim();
        __node_reclaim();
            shrink_node();

// 2. positive page out by kswapd
static int kswapd(void *p)
{
  unsigned int alloc_order, reclaim_order;
  unsigned int classzone_idx = MAX_NR_ZONES - 1;
  pg_data_t *pgdat = (pg_data_t*)p;
  struct task_struct *tsk = current;

    for ( ; ; ) {

        kswapd_try_to_sleep(pgdat, alloc_order, reclaim_order,
            classzone_idx);

        reclaim_order = balance_pgdat(pgdat, alloc_order, classzone_idx);

    }
}

balance_pgdat();
    kswapd_shrink_node();
        shrink_node();
            shrink_node_memcg();

```
![Linux Page State](../Images/linux-page-state.png)

```C++
// arch/x86/kernel/cpu/common.c
DEFINE_PER_CPU_PAGE_ALIGNED(struct gdt_page, gdt_page) = { .gdt = {
    [GDT_ENTRY_KERNEL32_CS]		= GDT_ENTRY_INIT(0xc09b, 0, 0xfffff),
    [GDT_ENTRY_KERNEL_CS]		= GDT_ENTRY_INIT(0xa09b, 0, 0xfffff),
    [GDT_ENTRY_KERNEL_DS]		= GDT_ENTRY_INIT(0xc093, 0, 0xfffff),
    [GDT_ENTRY_DEFAULT_USER32_CS]	= GDT_ENTRY_INIT(0xc0fb, 0, 0xfffff),
    [GDT_ENTRY_DEFAULT_USER_DS]	= GDT_ENTRY_INIT(0xc0f3, 0, 0xfffff),
    [GDT_ENTRY_DEFAULT_USER_CS]	= GDT_ENTRY_INIT(0xa0fb, 0, 0xfffff),
}};

#define __KERNEL_CS (GDT_ENTRY_KERNEL_CS*8)
#define __KERNEL_DS (GDT_ENTRY_KERNEL_DS*8)
#define __USER_DS (GDT_ENTRY_DEFAULT_USER_DS*8 + 3)
#define __USER_CS (GDT_ENTRY_DEFAULT_USER_CS*8 + 3)

```

# IPC
### PIPE
```C++
SYSCALL_DEFINE1(pipe, int __user *, fildes)
{
  return sys_pipe2(fildes, 0);
}

SYSCALL_DEFINE2(pipe2, int __user *, fildes, int, flags)
{
  struct file *files[2];
  int fd[2];
  int error;

  error = __do_pipe_flags(fd, files, flags);
  if (!error) {
    if (unlikely(copy_to_user(fildes, fd, sizeof(fd)))) {
      error = -EFAULT;
    } else {
      fd_install(fd[0], files[0]);
      fd_install(fd[1], files[1]);
    }
  }
  return error;
}


static int __do_pipe_flags(int *fd, struct file **files, int flags)
{
    int error;
    int fdw, fdr;
    error = create_pipe_files(files, flags);
    error = get_unused_fd_flags(flags);
    fdr = error;

    error = get_unused_fd_flags(flags);
    fdw = error;

    fd[0] = fdr;
    fd[1] = fdw;
    return 0;
}


int create_pipe_files(struct file **res, int flags)
{
    int err;
    struct inode *inode = get_pipe_inode();
    struct file *f;
    struct path path;
    path.dentry = d_alloc_pseudo(pipe_mnt->mnt_sb, &empty_name);
    path.mnt = mntget(pipe_mnt);

    d_instantiate(path.dentry, inode);

    f = alloc_file(&path, FMODE_WRITE, &pipefifo_fops);
    f->f_flags = O_WRONLY | (flags & (O_NONBLOCK | O_DIRECT));
    f->private_data = inode->i_pipe;

    res[0] = alloc_file(&path, FMODE_READ, &pipefifo_fops);
    path_get(&path);
    res[0]->private_data = inode->i_pipe;
    res[0]->f_flags = O_RDONLY | (flags & O_NONBLOCK);
    res[1] = f;
    return 0;
}

static struct file_system_type pipe_fs_type = {
  .name    = "pipefs",
  .mount    = pipefs_mount,
  .kill_sb  = kill_anon_super,
};

static int __init init_pipe_fs(void)
{
  int err = register_filesystem(&pipe_fs_type);

  if (!err) {
    pipe_mnt = kern_mount(&pipe_fs_type);
  }
}

static struct inode * get_pipe_inode(void)
{
  struct inode *inode = new_inode_pseudo(pipe_mnt->mnt_sb);
  struct pipe_inode_info *pipe;
  inode->i_ino = get_next_ino();

  pipe = alloc_pipe_info();
  inode->i_pipe = pipe;
  pipe->files = 2;
  pipe->readers = pipe->writers = 1;
  inode->i_fop = &pipefifo_fops;
  inode->i_state = I_DIRTY;
  inode->i_mode = S_IFIFO | S_IRUSR | S_IWUSR;
  inode->i_uid = current_fsuid();
  inode->i_gid = current_fsgid();
  inode->i_atime = inode->i_mtime = inode->i_ctime = current_time(inode);

  return inode;
}
```

### sem, shm, msg
```C++
struct ipc_namespace {
  struct ipc_ids  ids[3];
}

#define IPC_SEM_IDS  0
#define IPC_MSG_IDS  1
#define IPC_SHM_IDS  2

#define sem_ids(ns)  ((ns)->ids[IPC_SEM_IDS])
#define msg_ids(ns)  ((ns)->ids[IPC_MSG_IDS])
#define shm_ids(ns)  ((ns)->ids[IPC_SHM_IDS])


struct ipc_ids {
  int in_use;
  unsigned short seq;
  struct rw_semaphore rwsem;
  struct idr ipcs_idr;
  int next_id;
};

struct idr {
  struct radix_tree_root  idr_rt;
  unsigned int    idr_next;
};
```
![ipc_ids](../Images/ipc_ids.png)
```C++
struct kern_ipc_perm *ipc_obtain_object_idr(struct ipc_ids *ids, int id)
{
  struct kern_ipc_perm *out;
  int lid = ipcid_to_idx(id);
  out = idr_find(&ids->ipcs_idr, lid);
  return out;
}

static inline struct sem_array *sem_obtain_object(struct ipc_namespace *ns, int id)
{
  struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&sem_ids(ns), id);
  return container_of(ipcp, struct sem_array, sem_perm);
}

static inline struct msg_queue *msq_obtain_object(struct ipc_namespace *ns, int id)
{
  struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&msg_ids(ns), id);
  return container_of(ipcp, struct msg_queue, q_perm);
}

static inline struct shmid_kernel *shm_obtain_object(struct ipc_namespace *ns, int id)
{
  struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&shm_ids(ns), id);
  return container_of(ipcp, struct shmid_kernel, shm_perm);
}

SYSCALL_DEFINE3(shmget, key_t, key, size_t, size, int, shmflg)
{
  struct ipc_namespace *ns;
  static const struct ipc_ops shm_ops = {
    .getnew = newseg,
    .associate = shm_security,
    .more_checks = shm_more_checks,
  };
  struct ipc_params shm_params;
  ns = current->nsproxy->ipc_ns;
  shm_params.key = key;
  shm_params.flg = shmflg;
  shm_params.u.size = size;
  return ipcget(ns, &shm_ids(ns), &shm_ops, &shm_params);
}


int ipcget(struct ipc_namespace *ns, struct ipc_ids *ids,
    const struct ipc_ops *ops, struct ipc_params *params)
{
  if (params->key == IPC_PRIVATE)
    return ipcget_new(ns, ids, ops, params);
  else
    return ipcget_public(ns, ids, ops, params);
}

static int ipcget_public(struct ipc_namespace *ns, struct ipc_ids *ids,
    const struct ipc_ops *ops, struct ipc_params *params)
{
  struct kern_ipc_perm *ipcp;
  int flg = params->flg;
  int err;
  ipcp = ipc_findkey(ids, params->key);
  if (ipcp == NULL) {
    if (!(flg & IPC_CREAT))
      err = -ENOENT;
    else
      err = ops->getnew(ns, params);
  } else {
    if (flg & IPC_CREAT && flg & IPC_EXCL)
      err = -EEXIST;
    else {
      err = 0;
      if (ops->more_checks)
        err = ops->more_checks(ipcp, params);
    }
  }
  return err;
}


static int newseg(struct ipc_namespace *ns, struct ipc_params *params)
{
  key_t key = params->key;
  int shmflg = params->flg;
  size_t size = params->u.size;
  int error;
  struct shmid_kernel *shp;
  size_t numpages = (size + PAGE_SIZE - 1) >> PAGE_SHIFT;
  struct file *file;
  char name[13];
  vm_flags_t acctflag = 0;

  shp = kvmalloc(sizeof(*shp), GFP_KERNEL);
  shp->shm_perm.key = key;
  shp->shm_perm.mode = (shmflg & S_IRWXUGO);
  shp->mlock_user = NULL;

  shp->shm_perm.security = NULL;
  file = shmem_kernel_file_setup(name, size, acctflag);
  shp->shm_cprid = task_tgid_vnr(current);
  shp->shm_lprid = 0;
  shp->shm_atim = shp->shm_dtim = 0;
  shp->shm_ctim = get_seconds();
  shp->shm_segsz = size;
  shp->shm_nattch = 0;
  shp->shm_file = file;
  shp->shm_creator = current;

  error = ipc_addid(&shm_ids(ns), &shp->shm_perm, ns->shm_ctlmni);
  list_add(&shp->shm_clist, &current->sysvshm.shm_clist);
  file_inode(file)->i_ino = shp->shm_perm.id;

  ns->shm_tot += numpages;
  error = shp->shm_perm.id;
  return error;
}

int __init shmem_init(void)
{
  int error;
  error = shmem_init_inodecache();
  error = register_filesystem(&shmem_fs_type);
  shm_mnt = kern_mount(&shmem_fs_type);
  return 0;
}

static struct file_system_type shmem_fs_type = {
  .owner    = THIS_MODULE,
  .name    = "tmpfs",
  .mount    = shmem_mount,
  .kill_sb  = kill_litter_super,
  .fs_flags  = FS_USERNS_MOUNT,
};

struct file *shmem_kernel_file_setup(const char *name, loff_t size, unsigned long flags)
{
  return __shmem_file_setup(name, size, flags, S_PRIVATE);
}

static struct file *__shmem_file_setup(const char *name, loff_t size,
               unsigned long flags, unsigned int i_flags)
{
  struct file *res;
  struct inode *inode;
  struct path path;
  struct super_block *sb;
  struct qstr this;

  this.name = name;
  this.len = strlen(name);
  this.hash = 0; /* will go */
  sb = shm_mnt->mnt_sb;
  path.mnt = mntget(shm_mnt);
  path.dentry = d_alloc_pseudo(sb, &this);
  d_set_d_op(path.dentry, &anon_ops);

  inode = shmem_get_inode(sb, NULL, S_IFREG | S_IRWXUGO, 0, flags);
  inode->i_flags |= i_flags;
  d_instantiate(path.dentry, inode);
  inode->i_size = size;

  res = alloc_file(&path, FMODE_WRITE | FMODE_READ,
      &shmem_file_operations);
  return res;
}

static const struct file_operations shmem_file_operations = {
  .mmap    = shmem_mmap,
  .get_unmapped_area = shmem_get_unmapped_area,
#ifdef CONFIG_TMPFS
  .llseek    = shmem_file_llseek,
  .read_iter  = shmem_file_read_iter,
  .write_iter  = generic_file_write_iter,
  .fsync    = noop_fsync,
  .splice_read  = generic_file_splice_read,
  .splice_write  = iter_file_splice_write,
  .fallocate  = shmem_fallocate,
#endif
};


SYSCALL_DEFINE3(shmat, int, shmid, char __user *, shmaddr, int, shmflg)
{
    unsigned long ret;
    long err;
    err = do_shmat(shmid, shmaddr, shmflg, &ret, SHMLBA);
    force_successful_syscall_return();
    return (long)ret;
}

long do_shmat(int shmid, char __user *shmaddr, int shmflg,
        ulong *raddr, unsigned long shmlba)
{
  struct shmid_kernel *shp;
  unsigned long addr = (unsigned long)shmaddr;
  unsigned long size;
  struct file *file;
  int    err;
  unsigned long flags = MAP_SHARED;
  unsigned long prot;
  int acc_mode;
  struct ipc_namespace *ns;
  struct shm_file_data *sfd;
  struct path path;
  fmode_t f_mode;
  unsigned long populate = 0;

  prot = PROT_READ | PROT_WRITE;
  acc_mode = S_IRUGO | S_IWUGO;
  f_mode = FMODE_READ | FMODE_WRITE;

  ns = current->nsproxy->ipc_ns;
  shp = shm_obtain_object_check(ns, shmid);

  path = shp->shm_file->f_path;
  path_get(&path);
  shp->shm_nattch++;
  size = i_size_read(d_inode(path.dentry));

  sfd = kzalloc(sizeof(*sfd), GFP_KERNEL);

  file = alloc_file(&path, f_mode,
        is_file_hugepages(shp->shm_file) ?
        &shm_file_operations_huge :
        &shm_file_operations);

  file->private_data = sfd;
  file->f_mapping = shp->shm_file->f_mapping;
  sfd->id = shp->shm_perm.id;
  sfd->ns = get_ipc_ns(ns);
  sfd->file = shp->shm_file;
  sfd->vm_ops = NULL;

  addr = do_mmap_pgoff(file, addr, size, prot, flags, 0, &populate, NULL);
  *raddr = addr;
  err = 0;

  return err;
}


static int shm_mmap(struct file *file, struct vm_area_struct *vma)
{
  struct shm_file_data *sfd = shm_file_data(file);
  int ret;
  ret = __shm_open(vma);
  ret = call_mmap(sfd->file, vma);
  sfd->vm_ops = vma->vm_ops;
  vma->vm_ops = &shm_vm_ops;
  return 0;
}

static int shmem_mmap(struct file *file, struct vm_area_struct *vma)
{
  file_accessed(file);
  vma->vm_ops = &shmem_vm_ops;
  return 0;
}

static const struct vm_operations_struct shm_vm_ops = {
  .open  = shm_open,  /* callback for a new vm-area open */
  .close  = shm_close,  /* callback for when the vm-area is released */
  .fault  = shm_fault,
};

static const struct vm_operations_struct shmem_vm_ops = {
  .fault    = shmem_fault,
  .map_pages  = filemap_map_pages,
};

static int shm_fault(struct vm_fault *vmf)
{
  struct file *file = vmf->vma->vm_file;
  struct shm_file_data *sfd = shm_file_data(file);
  return sfd->vm_ops->fault(vmf);
}

static int shmem_fault(struct vm_fault *vmf)
{
  struct vm_area_struct *vma = vmf->vma;
  struct inode *inode = file_inode(vma->vm_file);
  gfp_t gfp = mapping_gfp_mask(inode->i_mapping);

  error = shmem_getpage_gfp(inode, vmf->pgoff, &vmf->page, sgp,
          gfp, vma, vmf, &ret);
}

static int shmem_getpage_gfp(struct inode *inode, pgoff_t index,
  struct page **pagep, enum sgp_type sgp, gfp_t gfp,
  struct vm_area_struct *vma, struct vm_fault *vmf, int *fault_type)
{
  page = shmem_alloc_and_acct_page(gfp, info, sbinfo,
        index, false);
}
```
![sem-shm-msg.png](../Images/sem-shm-msg.png)


# File Management
![linux-vfs-system.png](../Images/linux-vfs-system.png)

### inode, extents
```C++
struct ext4_inode {
  __le16  i_mode;    /* File mode */
  __le16  i_uid;    /* Low 16 bits of Owner Uid */
  __le32  i_size_lo;  /* Size in bytes */
  __le32  i_atime;  /* Access time */
  __le32  i_ctime;  /* Inode Change time */
  __le32  i_mtime;  /* Modification time */
  __le32  i_dtime;  /* Deletion Time */
  __le16  i_gid;    /* Low 16 bits of Group Id */
  __le16  i_links_count;  /* Links count */
  __le32  i_blocks_lo;  /* Blocks count */
  __le32  i_flags;  /* File flags */

  __le32  i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
  __le32  i_generation;  /* File version (for NFS) */
  __le32  i_file_acl_lo;  /* File ACL */
  __le32  i_size_high;

};

// Each block (leaves and indexes), even inode-stored has header.
struct ext4_extent_header {
  __le16  eh_magic;  /* probably will support different formats */
  __le16  eh_entries;  /* number of valid entries */
  __le16  eh_max;    /* capacity of store in entries */
  __le16  eh_depth;  /* has tree real underlying blocks? */
  __le32  eh_generation;  /* generation of the tree */
};

struct ext4_extent_idx {
  __le32  ei_block;  /* index covers logical blocks from 'block' */
  __le32  ei_leaf_lo;  /* pointer to the physical block of the next *
         * level. leaf or next index could be there */
  __le16  ei_leaf_hi;  /* high 16 bits of physical block */
  __u16  ei_unused;
};

struct ext4_extent {
  __le32  ee_block;  /* first logical block extent covers */
  // the most significant bit used as a flag to identify whether
  // this entent is initialized, 15 bits can present max 128MB data
  __le16  ee_len;    /* number of blocks covered by extent */
  __le16  ee_start_hi;  /* high 16 bits of physical block */
  __le32  ee_start_lo;  /* low 32 bits of physical block */
};
```
![linux-extents.jpg](../Images/linux-extents.jpg)

![linux-ext4-extents.png](../Images/linux-ext4-extents.png)

```C++
const struct inode_operations ext4_dir_inode_operations = {
  .create    = ext4_create,
  .lookup    = ext4_lookup,
  .link    = ext4_link,
  .unlink    = ext4_unlink,
  .symlink  = ext4_symlink,
  .mkdir    = ext4_mkdir,
  .rmdir    = ext4_rmdir,
  .mknod    = ext4_mknod,
  .tmpfile  = ext4_tmpfile,
  .rename    = ext4_rename2,
  .setattr  = ext4_setattr,
  .getattr  = ext4_getattr,
  .listxattr  = ext4_listxattr,
  .get_acl  = ext4_get_acl,
  .set_acl  = ext4_set_acl,
  .fiemap         = ext4_fiemap,
};

// ext4_create->ext4_new_inode_start_handle->__ext4_new_inode
struct inode *__ext4_new_inode(handle_t *handle, struct inode *dir,
             umode_t mode, const struct qstr *qstr,
             __u32 goal, uid_t *owner, __u32 i_flags,
             int handle_type, unsigned int line_no,
             int nblocks)
{
inode_bitmap_bh = ext4_read_inode_bitmap(sb, group);
ino = ext4_find_next_zero_bit((unsigned long *)
                inode_bitmap_bh->b_data,
                EXT4_INODES_PER_GROUP(sb), ino);
}
```

### Meta Block Group
```C++
struct ext4_group_desc
{
  __le32	bg_block_bitmap_lo;	/* Blocks bitmap block */
  __le32	bg_inode_bitmap_lo;	/* Inodes bitmap block */
  __le32	bg_inode_table_lo;	/* Inodes table block */
};
```
![linux-block-group.jpg](../Images/linux-block-group.jpg)

```C++
struct ext4_super_block {
  __le32  s_blocks_count_lo;  /* Blocks count */
  __le32  s_r_blocks_count_lo;  /* Reserved blocks count */
  __le32  s_free_blocks_count_lo;  /* Free blocks count */

  __le32  s_blocks_count_hi;  /* Blocks count */
  __le32  s_r_blocks_count_hi;  /* Reserved blocks count */
  __le32  s_free_blocks_count_hi;  /* Free blocks count */
}
```
![linux-meta-block-group.jpg](../Images/linux-meta-block-group.jpg)

### Directory
```C++
struct ext4_dir_entry {
  __le32  inode;      /* Inode number */
  __le16  rec_len;    /* Directory entry length */
  __le16  name_len;    /* Name length */
  char  name[EXT4_NAME_LEN];  /* File name */
};
struct ext4_dir_entry_2 {
  __le32  inode;      /* Inode number */
  __le16  rec_len;    /* Directory entry length */
  __u8  name_len;    /* Name length */
  __u8  file_type;
  char  name[EXT4_NAME_LEN];  /* File name */
};

struct dx_root
{
  struct fake_dirent dot;
  char dot_name[4];
  struct fake_dirent dotdot;
  char dotdot_name[4];
  struct dx_root_info
  {
    __le32 reserved_zero;
    u8 hash_version;
    u8 info_length; /* 8 */
    u8 indirect_levels;
    u8 unused_flags;
  }
  info;
  struct dx_entry  entries[0];
};

struct dx_entry
{
  __le32 hash;
  __le32 block;
};
```
![linux-directory.jpg](../Images/linux-directory.jpg)

### hard/symbolic link
```C++
 ln [args] [dst] [src]
```
![linux-link.jpg](../Images/linux-link.jpg)

![linux-file-system.png](../Images/linux-file-system.png)

### vfs
![linux-vfs.jpg](../Images/linux-vfs.jpg)
```C++
register_filesystem(&ext4_fs_type);

static struct file_system_type ext4_fs_type = {
  .owner    = THIS_MODULE,
  .name    = "ext4",
  .mount    = ext4_mount,
  .kill_sb  = kill_block_super,
  .fs_flags  = FS_REQUIRES_DEV,
};
```

#### mount
```C++

SYSCALL_DEFINE5(mount, char __user *, dev_name, char __user *, dir_name, char __user *, type, unsigned long, flags, void __user *, data)
{
  ret = do_mount(kernel_dev, dir_name, kernel_type, flags, options);
}
// do_mount->do_new_mount->vfs_kern_mount

struct vfsmount *
vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data)
{
  mnt = alloc_vfsmnt(name);
  root = mount_fs(type, flags, name, data);

  mnt->mnt.mnt_root = root;
  mnt->mnt.mnt_sb = root->d_sb;
  mnt->mnt_mountpoint = mnt->mnt.mnt_root;
  mnt->mnt_parent = mnt;
  list_add_tail(&mnt->mnt_instance, &root->d_sb->s_mounts);
  return &mnt->mnt;
}

struct mount {
  struct hlist_node mnt_hash;
  struct mount *mnt_parent;
  struct dentry *mnt_mountpoint;
  struct vfsmount mnt;
  union {
    struct rcu_head mnt_rcu;
    struct llist_node mnt_llist;
  };
  struct list_head mnt_mounts;  /* list of children, anchored here */
  struct list_head mnt_child;  /* and going through their mnt_child */
  struct list_head mnt_instance;  /* mount instance on sb->s_mounts */
  const char *mnt_devname;  /* Name of device e.g. /dev/dsk/hda1 */
  struct list_head mnt_list;
} __randomize_layout;

struct vfsmount {
  struct dentry *mnt_root;  /* root of the mounted tree */
  struct super_block *mnt_sb;  /* pointer to superblock */
  int mnt_flags;
} __randomize_layout;

struct dentry *
mount_fs(struct file_system_type *type, int flags, const char *name, void *data)
{
  struct dentry *root;
  struct super_block *sb;
  root = type->mount(type, flags, name, data);
  sb = root->d_sb;
}
```
![linux-mount-example.jpg](../Images/linux-mount-example.jpg)

```C++
struct file {
  union {
      struct llist_node	fu_llist;
      struct rcu_head 	fu_rcuhead;
  } f_u;
  struct path		f_path;
  struct inode		*f_inode;	/* cached value */
  const struct file_operations	*f_op;

  spinlock_t		                f_lock;
  enum rw_hint		              f_write_hint;
  atomic_long_t		              f_count;
  unsigned int 		              f_flags;
  fmode_t			                  f_mode;
  loff_t                          f_pos;
  struct mutex		              f_pos_lock;
  struct fown_struct	          f_owner;

#endif /* #ifdef CONFIG_EPOLL */
  struct address_space	*f_mapping;
  errseq_t		f_wb_err;
};

struct path {
  struct vfsmount *mnt;
  struct dentry *dentry;
};

// memroy chache of dirctories and files
struct dentry {
  unsigned int d_flags;		/* protected by d_lock */
  struct dentry *d_parent;	/* parent directory */
  struct qstr d_name;
  struct inode *d_inode;		/* Where the name belongs to - NULL is
           * negative */
  unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */

  const struct dentry_operations *d_op;
  struct super_block *d_sb;	/* The root of the dentry tree */

  struct hlist_bl_node d_hash;	/* lookup hash list */
  union {
    struct list_head d_lru;		/* LRU list */
    wait_queue_head_t *d_wait;	/* in-lookup ones only */
  };
  struct list_head d_child;	/* child of parent list */
  struct list_head d_subdirs;	/* our children */
} __randomize_layout;
```

### open
```C++
struct task_struct {
  struct fs_struct    *fs;
  struct files_struct *files;
  struct nsproxy  *nsproxy;
}

struct files_struct {
  struct file __rcu * fd_array[NR_OPEN_DEFAULT];
};

SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
  return do_sys_open(AT_FDCWD, filename, flags, mode);
}

long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
{
  fd = get_unused_fd_flags(flags);
  if (fd >= 0) {
    struct file *f = do_filp_open(dfd, tmp, &op);
    if (IS_ERR(f)) {
      put_unused_fd(fd);
      fd = PTR_ERR(f);
    } else {
      fsnotify_open(f);
      fd_install(fd, f);
    }
  }
  putname(tmp);
  return fd;
}

struct file *do_filp_open(int dfd, struct filename *pathname,
    const struct open_flags *op)
{
  set_nameidata(&nd, dfd, pathname);
  filp = path_openat(&nd, op, flags | LOOKUP_RCU);
  restore_nameidata();
  return filp;
}


static struct file *path_openat(struct nameidata *nd,
      const struct open_flags *op, unsigned flags)
{
  file = get_empty_filp();
  s = path_init(nd, flags);
  while (!(error = link_path_walk(s, nd)) &&
    (error = do_last(nd, file, op, &opened)) > 0) {
  }
  terminate_walk(nd);
  return file;
}

static int do_last(struct nameidata *nd,
       struct file *file, const struct open_flags *op,
       int *opened)
{
  error = lookup_fast(nd, &path, &inode, &seq); // loopup in dcache
  error = lookup_open(nd, &path, file, op, got_write, opened);
  error = vfs_open(&nd->path, file, current_cred());
}

static int lookup_open(struct nameidata *nd, struct path *path,
  struct file *file,
  const struct open_flags *op,
  bool got_write, int *opened)
{

  if (!dentry->d_inode && (open_flag & O_CREAT)) {
    error = dir_inode->i_op->create(dir_inode, dentry, mode,
            open_flag & O_EXCL);
  }
}


static int lookup_open(struct nameidata *nd, struct path *path,
      struct file *file,
      const struct open_flags *op,
      bool got_write, int *opened)
{

    dentry = d_alloc_parallel(dir, &nd->last, &wq);
    struct dentry *res = dir_inode->i_op->lookup(dir_inode, dentry,
                   nd->flags);
    path->dentry = dentry;
  path->mnt = nd->path.mnt;
}

const struct inode_operations ext4_dir_inode_operations = {
  .create    = ext4_create,
  .lookup    = ext4_lookup
}

int vfs_open(const struct path *path, struct file *file,
  const struct cred *cred)
{
  struct dentry *dentry = d_real(path->dentry, NULL, file->f_flags, 0);
  file->f_path = *path;
  return do_dentry_open(file, d_backing_inode(dentry), NULL, cred);
}

static int do_dentry_open(struct file *f,
  struct inode *inode,
  int (*open)(struct inode *, struct file *),
  const struct cred *cred)
{
  f->f_mode = OPEN_FMODE(f->f_flags) | FMODE_LSEEK |
        FMODE_PREAD | FMODE_PWRITE;
  path_get(&f->f_path);
  f->f_inode = inode;
  f->f_mapping = inode->i_mapping;
  f->f_op = fops_get(inode->i_fop);
  open = f->f_op->open;
  error = open(inode, f);
  f->f_flags &= ~(O_CREAT | O_EXCL | O_NOCTTY | O_TRUNC);
  file_ra_state_init(&f->f_ra, f->f_mapping->host->i_mapping);
  return 0;
}


const struct file_operations ext4_file_operations = {
  .open    = ext4_file_open,
};

```
![linux-dcache.jpg](../Images/linux-dcache.jpg)

### Question:
1. How to use inode bit map present all inodes?
2. Does system alloc a block for a dirctory or a file?