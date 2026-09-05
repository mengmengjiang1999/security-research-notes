
# JailHouse

先起一个Linux，Linux里面有一个Jailhouse命令行工具。然后命令行工具会打开/dev/jailhouse设备，open这个设备会返回一个fd。通过ioctl进内核态。执行driver里的handler（就是文档里面写好的这些）从jailhouse_cmd_enable()开始。

### driver

1. 	将hypervisor加载到内存里
/* Load hypervisor image */
err = request_firmware(&hypervisor, fw_name, jailhouse_dev);


2. 给hypervisor分配一块物理地址，并且在页表中建立映射

/* Unmap hypervisor_mem from a previous "enable". The mapping has to be
	 * redone since the root-cell config might have changed. */
	jailhouse_firmware_free();

	hypervisor_mem_res = request_mem_region(hv_mem->phys_start,
						hv_mem->size,
						"Jailhouse hypervisor");

	/* Map physical memory region reserved for Jailhouse. */
	hypervisor_mem = jailhouse_ioremap(hv_mem->phys_start, remap_addr,
					   hv_mem->size);
	if (!hypervisor_mem) {
		pr_err("jailhouse: Unable to map RAM reserved for hypervisor "
		       "at %08lx\n", (unsigned long)hv_mem->phys_start);
		goto error_release_memreg;
	}

3. 把hypervisor的二进制文件复制到这块内存里，剩余部分初始化为0

	/* Copy hypervisor's binary image at beginning of the memory region
	 * and clear the rest to zero. */
	memcpy(hypervisor_mem, hypervisor->data, hypervisor->size);
	memset(hypervisor_mem + hypervisor->size, 0,
	       hv_mem->size - hypervisor->size);

4. /* Copy system configuration to its target address in hypervisor memory
	 * region. */

5. on_each_cpu(enter_hypervisor, header, 0);


### enter_hypervisor函数

1. 调用地址为entry的函数

`
	entry = header->entry + (unsigned long) hypervisor_mem;

	if (cpu < header->max_cpus)
		/* either returns 0 or the same error code across all CPUs */
		err = entry(cpu);

`

entry地址的那个函数是编译的时候确定的。实际上调用的是arch_entry函数，因为：

setup.c
`
.entry = arch_entry - JAILHOUSE_BASE,
`

### arch_entry函数：

1. 保存了一堆寄存器
	/* set up the stack and push the root cell's callee saved registers */
	add	sp, x1, #PERCPU_STACK_END
	stp	x29, x17, [sp, #-16]!	/* note: our caller lr is in x17 */
	stp	x27, x28, [sp, #-16]!
	stp	x25, x26, [sp, #-16]!
	stp	x23, x24, [sp, #-16]!
	stp	x21, x22, [sp, #-16]!
	stp	x19, x20, [sp, #-16]!

（具体就不看了，总之是把寄存器状态都保存到内核栈里了）

2. 然后调用entry函数
	/* Call entry(cpuid, struct per_cpu*). Should not return. */
	bl	entry
	b	.

### entry函数 at hypervisor/setup.c


四个主要流程：

init_early(cpu_id);（master only）
cpu_init(cpu_data);
init_late();（master only）
arch_cpu_activate_vmm();

（master就是第一个进入这个函数的CPU）

EPT：init_early+init_late

1. init_early

建立hypervisor的hPA的映射，但是除了virtual_console之外的page都会映射到空白页。

（这里修改的是EPT）保证Linux不会看到hypervisor的内存

```
	hyp_phys_start = system_config->hypervisor_memory.phys_start;
	hyp_phys_end = hyp_phys_start + system_config->hypervisor_memory.size;

	hv_page.virt_start = hyp_phys_start;
	hv_page.size = PAGE_SIZE;
	hv_page.flags = JAILHOUSE_MEM_READ;
	while (hv_page.virt_start < hyp_phys_end) {
		if (virtual_console &&
		    hv_page.virt_start == paging_hvirt2phys(&console))
			hv_page.phys_start = paging_hvirt2phys(&console);
		else
			hv_page.phys_start = paging_hvirt2phys(empty_page);
		error = arch_map_memory_region(&root_cell, &hv_page);
		if (error)
			return;
		hv_page.virt_start += PAGE_SIZE;
	}

```

2. init_late()

建立其他memory region从gPA到hPA的映射

```c
for_each_mem_region(mem, root_cell.config, n) {
		if (JAILHOUSE_MEMORY_IS_SUBPAGE(mem))
			error = mmio_subpage_register(&root_cell, mem);
		else
			error = arch_map_memory_region(&root_cell, mem);
		if (error)
			return;
	}
```


hPT：

很多个CPU，每个CPU共享的一部分映射，共享的是从hypervisor的hVA到hypervisor的hPA的映射。
共享的部分的映射（gPA==hPA的那部分）直接把同一个页表项放进不同的CPU的HVPT里就可以实现共享。（好主意。

```c
err = paging_create_hvpt_link(&cpu_data->pg_structs, JAILHOUSE_BASE);
```

完成之后会映射当前CPU有特定的CPU数据，这部分进行映射的时候只要保证和共享的隔开一定距离。

```c
	/* set up private mapping of per-CPU data structure */
	err = paging_create(&cpu_data->pg_structs, paging_hvirt2phys(cpu_data),
			    sizeof(*cpu_data), LOCAL_CPU_BASE,
			    PAGE_DEFAULT_FLAGS,
			    PAGING_NON_COHERENT | PAGING_HUGE);
```

err = arch_cpu_init(cpu_data);

这个函数就是把刚才保存在内核栈的寄存器状态copy到hypervisor的per-CPU stack。
此外还会把CPU特权级切换到EL2。



3. 最后是arch_cpu_activate_vmm，从host进入guest

在这个函数里面调用
svm_vmentry函数，假装自己实在处理一个vmexit，然后就按照处理完vmexit之后返回VM的流程进入guest。
