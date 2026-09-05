

分为driver和hyperenclave两部分。

# HyperEnclave

### 问题

1. 启动流程（确实很像jailhouse）

driver如何使能enclave：
int he_cmd_enable(void)函数里实现，会进入enter_hypervisor函数

从driver到hypervisor：

enter_hypervisor(void *info)函数中的  entry = header->entry + (unsigned long)hypervisor_mem;

我们看一下发现header->entry指向的是arch_entry，在hypervisor/src/arch/x86_64里面。

第一行指令是 cli（clear interrupt flag）,调用switch_stack，然后调用entry函数，在main.rs里

extern "sysv64" fn entry(cpu_id: usize, linux_sp: usize) -> i32

然后调用 fn main(cpu_id: usize, linux_sp: usize) -> HvResult

这个应该是入口函数。
```rust
fn main(cpu_id: usize, linux_sp: usize) -> HvResult {
    let cpu_data = PerCpu::from_id_mut(cpu_id);
    let online_cpus = HvHeader::get().online_cpus as usize;
    let is_primary = ENTERED_CPUS.fetch_add(1, Ordering::SeqCst) == 0;
    wait_for_other_completed(&ENTERED_CPUS, online_cpus)?;
    println!(
        "{} CPU {} entered.",
        if is_primary { "Primary" } else { "Secondary" },
        cpu_id
    );

    // 第一个执行到这里的cpu设置为primary，后面的都是secondary
    // primary cpu进行一些初始化
    if is_primary {
        primary_init_early()?;
    } else {
        wait_for_other_completed(&INIT_EARLY_OK, 1)?;
    }

    // cpu_data init这里干了什么？应该是恢复上下文 TODO
    cpu_data.init(cpu_id, linux_sp, &cell::ROOT_CELL)?;
    println!("CPU {} init OK.", cpu_id);
    INITED_CPUS.fetch_add(1, Ordering::SeqCst);
    wait_for_other_completed(&INITED_CPUS, online_cpus)?;

    // 全都恢复上下文之后执行primary_init_late
    if is_primary {
        primary_init_late()?;
    } else {
        wait_for_other_completed(&INIT_LATE_OK, 1)?;
    }

    // 最后进入vmm中，这个函数就是恢复寄存器然后vmlaunch
    cpu_data.activate_vmm()
}
```


2. 创建enclave的流程

通过hypercall来完成。对应的hypercall是enclave_create。
传入的参数是一个guest physical address，记录了config信息

```rust
pub(super) fn enclave_create(
        &self,
        config_ptr: GuestPtr<HvEnclDesc>,
    ) -> HyperCallResult<usize> {
        let now = Instant::now();
        let secs_gpaddr = config_ptr.as_guest_paddr()?;
        let secs = *GuestPtr::gpaddr_to_ref(&secs_gpaddr, false)?;
        info!("enclave_create({:#x?}): {:#x?}", config_ptr, secs);
        // **********
        let enclave = Enclave::new(secs_gpaddr, config_ptr.guest_vaddr(), secs)?;//就是new一堆东西
        ENCLAVE_MANAGER.add_enclave(enclave.clone())?;
        //ENCLAVE_MANAGER是一个public static，一看名字就知道是用来manage enclave的
        //就是把刚创建的enclave让manager知道
        // **********
        enclave.atomic_add_stats(EnclaveStatsId::Create, now.elapsed());  // 看起来应该是enclave有一个信息是记录对enclave的操作的
        Ok(0)
    }
```


3. 如何向enclave里面添加页面

传入的是一个guest physical address

```rust
    pub(super) fn enclave_add_page(
        &self,
        page_desc_ptr: GuestPtr<HvEnclNewPageDesc>,
    ) -> HyperCallResult<usize> {
        let now = Instant::now();
        let page_desc = page_desc_ptr.read()?;
        debug!("enclave_add_page({:#x?}): {:#x?}", page_desc_ptr, page_desc);
        let config_ptr = page_desc
            .config_address
            .as_guest_ptr_ns::<HvEnclDesc>(&self.gpt, self.privilege_level());
        // ******************
        let enclave = ENCLAVE_MANAGER.find_enclave(config_ptr.as_guest_paddr()?)?;
        enclave.add_page(&page_desc, &self.gpt)?;
        // ******************
        enclave.atomic_add_stats(EnclaveStatsId::AddPage, now.elapsed());

        Ok(0)
    }
```

```rust
pub fn add_page(
        self: &Arc<Self>,
        page_desc: &HvEnclNewPageDesc,
        gpt: &GuestPageTableImmut,
    ) -> HyperCallResult<usize> {
        if self.state.load(Ordering::SeqCst) != STATE_UNINIT {
            return hypercall_hv_err_result!(
                EBUSY,
                "Enclave::add_page(): enclave is already initialized"
            );
        }
        //enclave只允许在没有init的时候添加page

        if !page_desc.attr.contains(EnclPageAttributes::EADD) {
            return hypercall_hv_err_result!(
                EINVAL,
                format!(
                    "Enclave::add_page(): Bad page attributes {:#x?}",
                    page_desc.attr
                )
            );
        }
        // 这是什么意思？

        let gvaddr = page_desc.enclave_lin_addr as usize;
        if !is_aligned(gvaddr) {
            return hypercall_hv_err_result!(
                EINVAL,
                format!(
                    "Enclave::add_page(): enclave_lin_addr {:#x} is not aligned",
                    gvaddr
                )
            );
        }
        // 没有对齐
        if !self.elrange.contains(&gvaddr) {
            return hypercall_hv_err_result!(
                EINVAL,
                format!(
                    "Enclave::add_page(): enclave_lin_addr {:#x} is out of ELRANGE {:#x?}",
                    gvaddr, self.elrange
                )
            );
        }
        // 总之上面的是error的情况


        let gpaddr = page_desc.epc_page_pa as usize;
        let sec_info_ptr = page_desc
            .metadata
            .as_guest_ptr_ns::<SgxSecInfo>(gpt, PrivilegeLevel::Supervisor);
        let sec_info = sec_info_ptr.read()?;
        EpcmManager::add_page(gvaddr, gpaddr, &sec_info, self)?;// 这是什么？

        let gpt_flags = sec_info.into();

        if sec_info.page_type == SgxEnclPageType::TCS {//TCS是什么？看起来是某种特殊的page，先不管了
            let tcs_gpaddr = page_desc.source_address as _;
            let tcs: &SgxTcs = GuestPtr::gpaddr_to_ref(&tcs_gpaddr, false)?;
            if !tcs.validate_at_creation() {
                return hypercall_hv_err_result!(
                    EINVAL,
                    format!("Enclave::add_page(): Invalid TCS: {:#x?}", tcs)
                );
            }
            info!("New enclave thread(tcs_vaddr={:#x}): {:#x?}", gvaddr, tcs);
            self.tcs_count.fetch_add(1, Ordering::Release);
        } else {//如果是一个普通的page，现在已经知道的是guest physical address
            let hpaddr = gpaddr; //采用恒等映射
            let npt_flags = gpt_flags | MemFlags::ENCRYPTED;
            self.npt.write().map(&MemoryRegion::new_with_offset_mapper(
                gpaddr, hpaddr, PAGE_SIZE, npt_flags,
            ))?;//写嵌套页表
        }
        //写guest页表
        self.gpt.write().map(&MemoryRegion::new_with_offset_mapper(
            gvaddr, gpaddr, PAGE_SIZE, gpt_flags,
        ))?;
        unsafe {//把page的内容复制到guest physical address
            core::ptr::copy_nonoverlapping(
                phys_to_virt(page_desc.source_address as _) as *const u8,
                phys_to_virt(gpaddr) as *mut u8,
                PAGE_SIZE,
            );
        }

        let page_data = if page_desc.attr.contains(EnclPageAttributes::EEXTEND) {
            Some(unsafe { &*(phys_to_virt(gpaddr) as *mut [u8; PAGE_SIZE]) })
        } else {
            None
        };
        self.measure
            .write()
            .update((gvaddr - self.elrange.start) as _, sec_info, page_data);
        //每进行一个hypercall都要更新一下measure，measure每次被更新都会重新计算enclave的sha256
        Ok(0)
    }
```

4. enclave里面存了哪些数据，需要什么数据结构来维护enclave的信息？

- ENCLAVE_MANAGER

- ENCLAVE

```rust
        let enclave = Arc::new(Self {
            id: secs_paddr,//guest physical address
            secs_vaddr,//guest virtual address
            secs: UnsafeCell::new(secs_verified),//
            elrange,/// Enclave Linear Address Range (ELRANGE).
            state: AtomicUsize::new(STATE_UNINIT),//记录enclave state
            /// Enclave state
            // const STATE_UNINIT: usize = 0x0;
            // const STATE_INIT_TRY: usize = 0x1;
            // const STATE_INIT_OK: usize = 0x2;
            // const STATE_TRY_DESTROY: usize = 0x3;
            // const STATE_IN_DESTROY: usize = 0x4;
            measure: RwLock::new(measure),
            // 这是用来计算enclave的sha256的
            npt,//页表
            gpt,//页表
            epc_page_num: AtomicIsize::new(0),//epc page number
            tcs_count: AtomicUsize::new(0),//number of tcs pages，这是什么？

            stats: Default::default(),// Statistics of enclave operation time.，每次对enclave进行修改都会记录：enclave.atomic_add_stats
            tracking_state: RwLock::new(Default::default()),// Tracking cycle state.
            encl_mem_lock: SpinMutex::new(()),
            shmem: RwLock::new(IntervalTree::new()),
            shmem_lock: RwLock::new(()),
            shmem_invalidating_cnt: AtomicIsize::new(0),
        });
```