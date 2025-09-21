---
layout: post
title:  "Spelunking through the linux kernel as a mentee"
date:   2025-09-20 13:34:32 -0700
categories: jekyll update
---
In March 2025, I joined the Linux Foundation's Linux Kernel Mentorship Program. As I will detail in this post, I fell down a deep rabbit hole rather early in the program, and stubbornly refused to get out of it for a while. I learned a few things from my time down there, failed to get a patch accepted and nearly quit the program, but persevered thanks in no small part to our mentor, Shuah Khan.

Before I joined the program, I had already decided to focus my efforts on the AMDGPU driver, the only reason being that I had a Steam Deck OLED. I thought it would be interesting to try and improve the system in some way. There are probably better ways to go about this, but I did all of my kernel development on the Steam Deck with Fedora installed. I had never used Fedora before, and I'm not sure if this is available in Ubuntu, but the Fedora kernel spec file makes it easier to install all of the software required to build the kernel: https://docs.fedoraproject.org/en-US/quick-docs/kernel-build-custom/. These requirements are also listed here: https://www.kernel.org/doc/html/next/process/changes.html. Needless to say, this development environment is a bit unconventional, and I did run into some strange errors when trying to compile the kernel:
```
GCC: arch/x86/tools/insn_decoder_test: error: malformed line 5952845:
2_>:ffffffff81f042d0
```

Using objdump -d on the vmlinux artifact, I was able to see the line:
```
ffffffff81f042d0 <__pfx__RNCINvNtNtNtCsf5tcb0XGUW4_4core4iter8adapters3map12map_try_foldjNtCsagR6JbSOIa9_12drm_panic_qr7VersionuINtNtNtBa_3ops12control_flow11ControlFlowB10_ENcB10_0NCINvNvNtNtNtB8_6traits8iterator8Iterator4find5checkB10_NCNvMB12_B10_13from_segments0E0E0B12_>:

```

map_try_fold doesn't sound very C-like, so I thought it was probably related to the Rust code in the kernel. This was resolved for me by removing the existing version of Rust on my system and replacing it with the current version from the Rust website. I had other rabbit holes to jump down, so I didn't look too deeply into this problem.   

Syzkaller (https://github.com/google/syzkaller) was mentioned during our mentorship sessions. I had a look at some existing bugs for the dri and amd-gfx subsystems (https://syzkaller.appspot.com/upstream/s/dri, https://syzkaller.appspot.com/upstream/s/amd-gfx), but found very few. I decided to run syzkaller on my Steam Deck to see if I could find any unreported issues.

The first issue I ran into was this error after installing the kernel with CONFIG_KCOV enabled in the .config:
```
workqueue: Failed to create a rescuer kthread for wq "amdgpu-reset-dev": -EINTR
        [drm:amdgpu_reset_create_reset_domain [amdgpu]] *ERROR* Failed to allocate wq for amdgpu_reset_domain!
        amdgpu 0000:04:00.0: amdgpu: Fatal error during GPU init
        amdgpu 0000:04:00.0: probe with driver amdgpu failed with error -12
```

This could be an interesting issue to examine, but again, my goal was just to get syzkaller running. I brought this error up during one of our mentorship meetings, and our mentor suggested reading a bit about the kernel initialization sequence and kernel start-up parameters (KNL) here: https://thenewstack.io/demystifying-linux-kernel-initialization/. This was very helpful, and by enabling asynchronous probing of the amdgpu driver with the driver_async_probe=amdgpu KNL, I was able to resolve the issue.

The next problem was getting syzkaller to run on the Steam Deck. I followed the syzkaller instructions for fuzzing the kernel on an isolated host (https://github.com/google/syzkaller/blob/master/docs/linux/setup_linux-host_isolated.md). I also had to modify the syzkaller config to target only the subsystem I was interested in. I found a blog post series by Collabora which provided some tips on how to do this: https://www.collabora.com/news-and-blog/blog/2020/04/17/using-syzkaller-to-detect-programming-bugs-in-linux/. I ended up using a config that looked something like this, with some variations on certain testing runs:
```
{
	"target": "linux/amd64",
	"http": "127.0.0.1:56741",
	"sshkey" : "/path",
	"workdir": "/path",
	"kernel_obj": "/path",
	"kernel_src": "/path",
	"syzkaller": "/path",
	"sandbox": "setuid",
	"type": "isolated",
	"enable_syscalls": ["openat$drirender128", "ioctl$DRM_*", "close"],
	"disable_syscalls": ["ioctl$DRM_IOCTL_SYNCOBJ_*"],
	"reproduce": false,
	"vm": {
		"targets" : [ "10.0.0.1" ],
		"pstore": false,
		"target_dir" : "/path",
                "target_reboot" : true
	}
}
```

I disabled some syscalls since they kept popping up during fuzzing, seemed to have been reported already, and I wanted to look at something new. Finally I was able to find a few interesting bugs:
```
==================================================================
BUG: KASAN: slab-use-after-free in rb_next+0xda/0x160 lib/rbtree.c:505
Read of size 8 at addr ffff8881805085e0 by task kworker/u32:12/192
CPU: 3 UID: 0 PID: 192 Comm: kworker/u32:12 Not tainted 6.14.0-flowejam-+ #1
Hardware name: Valve Galileo/Galileo, BIOS F7G0112 08/01/2024

==================================================================
BUG: KASAN: slab-use-after-free in rb_set_parent_color include/linux/rbtree_augmented.h:191 [inline]
BUG: KASAN: slab-use-after-free in __rb_erase_augmented include/linux/rbtree_augmented.h:312 [inline]
BUG: KASAN: slab-use-after-free in rb_erase+0x157c/0x1b10 lib/rbtree.c:443
Write of size 8 at addr ffff88816414c5d0 by task syz.2.3004/12376
CPU: 7 UID: 65534 PID: 12376 Comm: syz.2.3004 Not tainted 6.14.0-flowejam-+ #1
Hardware name: Valve Galileo/Galileo, BIOS F7G0112 08/01/2024

==================================================================
BUG: KASAN: slab-use-after-free in __rb_erase_augmented include/linux/rbtree_augmented.h:259 [inline]
BUG: KASAN: slab-use-after-free in rb_erase+0xf5d/0x1b10 lib/rbtree.c:443
Read of size 8 at addr ffff88812ebcc5e0 by task syz.1.814/6553
CPU: 0 UID: 65534 PID: 6553 Comm: syz.1.814 Not tainted 6.14.0-flowejam-+ #1
Hardware name: Valve Galileo/Galileo, BIOS F7G0112 08/01/2024

==================================================================
BUG: KASAN: slab-use-after-free in drm_sched_entity_compare_before drivers/gpu/drm/scheduler/sched_main.c:147 [inline] [gpu_sched]
BUG: KASAN: slab-use-after-free in rb_add_cached include/linux/rbtree.h:174 [inline] [gpu_sched]
BUG: KASAN: slab-use-after-free in drm_sched_rq_update_fifo_locked+0x47b/0x540 drivers/gpu/drm/scheduler/sched_main.c:175 [gpu_sched]
Read of size 8 at addr ffff8881208445c8 by task syz.1.49115/146644
CPU: 7 UID: 65534 PID: 146644 Comm: syz.1.49115 Not tainted 6.14.0-flowejam-+ #1
Hardware name: Valve Galileo/Galileo, BIOS F7G0112 08/01/2024
```

The call trace in the KASAN reports for all of these issues eventually pointed me towards looking at the amdgpu_vm_fini_entities function (https://elixir.bootlin.com/linux/v6.16.8/source/drivers/gpu/drm/amd/amdgpu/amdgpu_vm.c#L526). I found the most helpful debugging method to be old fashioned printk debugging, with some added pieces of info like the CPU # and PID, like so:
```
pr_err("flowejam: calling %s: pid = %d: cpu = %d: line number: %d\n", __func__, current->pid, smp_processor_id(), __LINE__);
```

In the end, I made a change that would prevent stopped entities from being added to the run queue in drm_sched_rq_update_fifo_locked. drivers/gpu/drm/scheduler/sched_main.c provides the following details about entities & the run queue in the GPU scheduler:
```
* The GPU scheduler provides entities which allow userspace to push jobs
* into software queues... 

...
* 2. Each scheduler has multiple run queues with different priorities
*    (e.g., HIGH_HW,HIGH_SW, KERNEL, NORMAL)
* 3. Each scheduler run queue has a queue of entities to schedule
* 4. Entities themselves maintain a queue of jobs that will be scheduled on
*    the hardware.
```

A stopped entity "marks the entity as removed from the run queue and destined for termination". 

This change seemed to me to fix the issue, since it had prevented any further KASAN reports. I ran some tests using the igt-gpu-tools testing suite, and I didn't notice any regressions from the results. At last! I had fixed the issue, and my changes didn't seem to cause any problems. For sure, this patch would be accepted!

Not so fast! As it turned out, the patch wasn't accepted for reasons which I'll explain. First of all, I didn't have a reproducer, so it's hard to be certain that my patch was correct. Also, it was pointed out that since the issue might be a race condition, it could come down to timing, and that the issue should be fixed where the locking was supposed to happen. This is understandable to me - my patch didn't get to the root of the issue.

The kernel maintainers have very high standards, and endeavour to ensure correctness in all accepted patches. Coming up with a fix to a particular bug can be extremely challenging. It's not an uncommon occurrence to put in many hours of work on a patch only to have it rejected. I may sound very accepting of this now, but at the time, this was a crushing blow. 

I spent an embarrassing amount of time trying various fixes that I thought would surely fix this bug. I saw it as a test: _if I can't solve this issue, maybe I'm not cut out for this kernel dev stuff!_ The result of this test was ambiguous. I had _sort of_ fixed the issue, but my patch had been rejected, and I was running out of time in the program while other mentees seemed to be making steady progress. I had been working on this issue for too long during my evenings and weekends, and couldn't afford to invest more time learning enough to write a reproducer for the issue. This setback nearly caused me to quit kernel development entirely.  

Our mentor, Shuah Khan, had some encouraging words for me and suggested I look at other issues. At this point I had two accepted patches, both of them addressing simpler issues - adding documentation for one function, and removing another that wasn't being used. Even these "simpler" issues take some work to understand the related code, and may require various tests to be run. It's often not easy to get even a single changed line accepted in the mainline kernel! There can be a huge amount of functionality to understand before you can make that change. And you will still, more often than not, be asked by reviewers to explain your change, or to send another version with requested changes. 

It is no small feat to get a patch accepted. Some advice for potential mentees reading this though: take a look at past changes by mentees that received a frosty reception. Learn from the mailing lists, and you'll get a sense of changes to avoid. 

I eventually managed to get a total of five patches accepted. In retrospect, it would have been better to focus on the simpler issues first, and timebox the syzkaller bugs for a much shorter period. You need a certain amount of persistence to contribute to the linux kernel though. I learned a lot from stubbornly sticking with that issue: DRM GPU scheduler & AMDGPU driver details, debugging tools such as ftrace, kernel command-line parameters, how to set up syzkaller to run on your own hardware, using cscope, among many other topics. It also generated a lot of interesting [discussion](https://lore.kernel.org/all/aIGz2fzBILVQnDW8@lstrano-desk.jf.intel.com/), and a maintainer kindly added my name in the "Reported-by" tag on a [FIXME comment](https://lore.kernel.org/all/20250813085654.102504-2-phasta@kernel.org/). I have a hunch that fuzzing more [DRM subsystem syscalls with syzkaller](https://github.com/google/syzkaller/blob/master/sys/linux/dev_dri.txt) will uncover other interesting issues to be investigated... see you down the next rabbit hole!      


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
