## Go program execute process

### 简介
	本文从Golang的小程序反汇编得到汇编代码，从中分析Golang程序执行过程。
	
	Golang 程序

	`
		package main
		import "fmt"

		func main() {
			ch1 := make(chan int)
			go func() {
				fmt.Println("hello")
				ch1 <- 1
		}()
		<-ch1
		}

	`
	先通过golang build -gcflags "-N -l" （禁止内联）生成可执行程序，然后通过GDB跟踪。

### 详情
	//入口
	1.TEXT runtime·rt0_go(SB),NOSPLIT,$0
		get_tls(BX)
	       	LEAQ    runtime·g0(SB), CX
       		MOVQ    CX, g(BX)
	        LEAQ    runtime·m0(SB), AX

		// save m->g0 = g0
	        MOVQ    CX, m_g0(AX)
       		// save m0 to g0->m
	        MOVQ    AX, g_m(CX)

		CALL    runtime·check(SB)
		MOVL    16(SP), AX              // copy argc
	        MOVL    AX, 0(SP)
       		MOVQ    24(SP), AX              // copy argv
       		MOVQ    AX, 8(SP)
       		CALL    runtime·args(SB)
	        CALL    runtime·osinit(SB)
	        CALL    runtime·schedinit(SB)

		// create a new goroutine to start program
	        MOVQ    $runtime·mainPC(SB), AX         // entry
       		PUSHQ   AX   
	        PUSHQ   $0                      // arg size
       		CALL    runtime·newproc(SB)
	        POPQ    AX   
        	POPQ    AX   

      		// start this M
		// mstart shou never return
	        CALL    runtime·mstart(SB)


	//类型测试
	2 runtime.check()

	//
	3 runtime.args()

	//
	4 runtime.osinit()
		ncpu = getproccount() //获取CPU计算
		physHugePageSize = getHugePageSize() //获取大页大小
		
	//The bootstrap sequence is:
	// call osinit
	// call schedinit
	// make & queue new G
	// call runtime.mstart

	5 runtime.schedinit()
		_g_ := getg() //获取当前g的指针
		sched.maxmcount = 10000 //m的最大计数为1w
		tracebackinit()
		moduledateverify()
		stackinit() //初始化stack pool
		mallocinit() //内存管理初始化，主要是初始花mheap_，初始化arena。定义不同内存状态的申请
		mcommoninit(_g_.m) //m公共初始化
		cpuinit() //cpu初始化
		alginit() //算法初始化
		modulesinit() //
		typelinksinit()
		itabsinit()

		msigsave(_g_.m)
		
		goargs()
		goenvs()
		parsedebuggvars() //解析debug变量
		procresize(procs) //write barrier needs a P	

	6 runtime.mallocinit()
		mheap_.init() //堆初始化
		_g_ := getg()
		_g_.m.mcache = allocmacache()

	7 runtime.mcommoninit()
		mpreinit(mp) //m.gsignal初始化，其中新建g，同时g的stack需要申请。从系统内存申请，或者从stackpool申请，或者从heap（mheap_）申请。
		
	8 runtime.procresize(nproc int32) *p
		// initialize new P's
		for i := old; i < nprocs; i++ {
			pp := allp[i]
			if pp == nil {
				pp = new(p)
			}

			pp.init(i) //设置pp的状态为:_Pgcstop,pp.mcache = getg().m.mcache //或者pp.mcache = allocmcache()
			atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
		}

	9 runtime.main()
		lockOSThread()		
		if g.m != &m0 {
			throw("runtime.main not on m0")
		} //当前工作线程必须是m0	

		doInit(&runtime_inittask) //必须要在defer之前

		gcenable() //开启gc，启动bgsweep() bgscavenge协程

		doInit(&main_inittask)
		unlockOSThread()

		fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
		fn()

		if atomic.Load(&panicking) != 0 {
			gopark(nil, nil, waitReasonPanicWait, traceEvGoStop, 1)
		}

		exit(0)

	10 runtime.gcenable() 
		c := make(chan int, 2)
		go bgsweep(c)
		go bgscavenge(c)
		<-c
		<-c
		memstats.enablegc = true

	11 func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int)
		// 将当前goroutine设置为waiting状态，并且调用unlockf函数
		// 如果unlockf返回false，goroutine恢复
		// reason解释goroutine parked的原因
		// 它显示在堆栈跟踪和堆转储中
		
		mp := acquirem() //获取当前goroutine关联的m
		gp := mp.curg //获取m中正在运行的goroutine
		status := readgstatus(gp) //获取gp(goroutine)的状态
		mp.waitlock = lock
		mp.waitunlock = unlockf
		
		gp.waitreason = reason

		mp.waittraceev = traceEv
		mp.waittraceskip = traceskip
		
		releasem(mp)

		// func mcall(fn func(*g))
		// mcall从g切换到g0栈调用fn(g)，其中g是发起调用的goroutine。
		// mcall将g当前的PC/SP保存在g-sched以便以后恢复。
		// 由fn决定稍后的执行，通过记录g到数据结构中，从而后面可以调用ready(g)
		// 当g被重新调度时，mcall随后返回原始goroutine g。
		// fn绝对不能返回；通常通过调用schedule结束，让m来执行其他goroutine。
		// 
		// mcall只能在g stack(非g0或者gsignal)中被调用
		//
		// 函数必须是 go:noescape(不能逃逸)，如果fn是栈分配的闭包，
		// fn将g放在运行队列中，g在fn返回之前执行。闭包在执行过程中无效。

		mcall(park_m)

		
	12 runtime.newproc(siz int32, fn *funcval) 
		//创建一个新的协程用于运行fn
		//将fn放在g的队列中等待运行
		//编译器将go声明转换成对它的调用
		//协程stack不能split因为它猜测函数参数在fn后面有效排列，如果stack split，他们不会复制

		argp := add(unsafe.Pointer(&fn), sys.PtrSize) //获取参数地址
		gp := getg() //获取当前goroutine
		pc := getcallerpc()

		//systemstack: 
		systemstack(func() {
			newproc1(fn, (*uint8)(argp), siz, gp, pc)
		}) 

		
	13 runtime.park_m
		// 在g0中继续暂停
		
		_g_ := getg()
		
		//将gp状态从_Grunning变更为_Gwaiting
		casgstatus(gp, _Grunning, _Gwaiting)

		//dropg 解除m与curg的联系
		dropg()

		if fn := _g_.m.waitunlockf; fn != nil {
			ok := fn(gp, _g_.m.waitlock)
			_g_.m.waitunlockf = nil
			_g_.m.waitlock = nil
			if !ok {
				casgstatus(gp, _Gwaiting, _Grunnable)
				execute(gp, true)
			}
		}

		schedule()

	14 runtime.casgstatus(gp *g, oldval, newval uint32)
		// 如果设置为Gscanstatus或者从Gscanstatus状态变更，这样抛出异常。取而代之的是castogscanstatus 和casfrom_Gscanstatus
	        // 如果g->atomicstatus处于Gscan状态，casgstatus将会一直循环，直到设置状态为Gscan的routine完成。
		
		if (oldval&_Gscan != 0) || (newval&_Gscan != 0) || oldval == newval {
			//throw() //在系统栈
		}	

		if oldval == _Grunning && gp.gcscanvalid {
			//throw() //在系统栈
		}

		const yieldDelay = 5 * 1000

		// 如果gp->atomicstatus处于扫描状态，循环会给GC时间完成并且将状态修改为oldval
		for i := 0; !atomic.Cas(&gp.atomicstatus, oldval, newval); i++ {
			if i == 0 {
				nextYield = nanotime() + yieldDelay
			}
			if nanotime() < nextYield {
				for x := 0; x < 10 && gp.atomicstatus != oldval; x++ {
					procyield(1)
				}
			} else {
				osyield()
				nextYield = nanotime() + yieldDelay/2
			}
		}	
		
		if newval == _Grunning {
			gp.gcscanvalid = false
		}

	15 runtime.dropg()
		// dropg将移除m和当前routine m->curg的关联
		// 通常调用者变更gp的Grunning状态然后立刻调用dropg完成该项工作
		// 调用者还负责安排gp在恰当的时间状态变更为ready重新启动。
		// 在调用dropg和稍后安排gp状态变更为ready，调用者可以做其他工作
		// 但最终应该调用schedule来重新启动这个m上面的goroutine调度。

		_g_ := getg()

		 setMNoWB(&_g_.m.curg.m, nil) //非写屏障设置m当前gouroutine的m为空
		 setGNoWB(&_g_.m.curg, nil) //非写屏障设置m当前goroutine为空

	16 runtime.execute(gp *g, inheritTime bool)
		// 调度gp在当前M上运行。
		// 如果inheritTime是true，gp在当前时间片中继承保留的时间。否则它开始一个新的
		// 时间片。
		// 从不返回

		// 允许写屏障因为在几个地方获得P之后立即调用。
		_g_ := getg()
		// 将gp的状态从_Grunnable变更为_Grunning
		casgstatus(gp, _Grunnable, _Grunning)
		gp.waitsince = 0
		gp.preemt = false
		gp.stackguard0 = gp.stack.lo + _StackGuard
		if !inheritTime {
			_g_.m.p.ptr().schedtick++
		}

		_g_.m.curg = gp //将m的正在运行的goroutine变更为gp
		gp.m = _g_.m //将gp的m变更为_g_.m

		// 检查profiler是否需要打开或者关闭
		hz := sched.profilehz
		if _g_.m.profilehz != hz {
			setThreadCPUProfiler(hz)
		}

		gogo(&gp.sched)

	17 runtime.schedule()
		// 调用执行一轮是：寻找一个可执行的goroutine并且执行。
		_g_ := getg()

		// guintptr, muintptr, and puintptr 用来绕过写屏障
		// 当 当前P被释放时，避免写屏障特别重要。因为GC认为整个
		// 程序已经停止，一个未预期的写屏障不会与GC同步，这可能导致
		// 对象已经标记但没有入队的执行到一半的写屏障。如果在入队之前
		// GC跳过该对象并且完成，这将导致不正确的释放该对象。

		// 我们尝试当没有拥有一个正在运行的P时只使用特殊的赋值函数，但是
		// 一些特殊内存单词的更新可以穿过写屏障，而有些没有。这打破写屏障阴影检查
		// 的模式，同时令人害怕的是：拥有一个完全被GC忽视的单词要比拥有一个只忽略少量
		// 更新的单词要好。

		// 在allgs和allp 列表中 或者来自栈变量（在他们到达那些列表之前的变量申请）的Gs
		// 和Ps通过正确的指针访问总是有效的。

		// 不论来自allm还是来自freem，Ms通过正确的指针访问总是有效。不像 Gs 和 Ps，我们
		// 释放 Ms，所有没有任何东西拥有越过安全点的muintptr非常重要。

		// A guintptr持有一个goroutine的指针，但是作为绕过写屏障的uintptr类型。它用在
		// Gobuf goroutine状态和不使用P操作的调度列表中。

		// Gobuf.g goroutine指针 总是被汇编代码更新。在一小部分地方中，
		// 它是由go代码更新 -func save-，它必须被看做一个uintptr为了避免在错误的时间
		// 产生写屏障。为了取代理解在非汇编操作中产生写屏障，我们改变uintptr的类型，所以
		// 它根本不需要写屏障

		// goroutine结构被发布在allg列表中并且从不释放。它将阻止goroutine结构被回收。
		// Gobuf.g 从来不是goroutine的唯一引用。在allg上发布的goroutine最新生成。
		// Goroutine的指针同时会保留在GC不可见的地方，比如：TLS，所以我不能看到它们
		// 移动过。如果我们确实想开始移动数据到GC，我们需要从恰当的arena中申请
		// goroutine结构。用guintptr指针不让错误变得更糟糕。

		if _g_.m.lockedg != 0 {
			stoplockedm()
			execute(_g_.m.lockedg.ptr(), false)
		}

	18 runtime.stoplockedm() 
		// 停止锁定到g的当前M 的执行，直到g再次执行。
		_g_ := getg()
		
		if _g_.m.p != 0 {
			// 调度其他M来运行这个P
			_p_ := releasep()
			handoffp(_p_)
		}

	19 runtime.releasep() *p
		// 断开p与当前m的联系
		_g_ := getg()
		// 获得当前m关联的p
		_p_ := _g_.m.p.ptr()
		// 当前m的p为0
		_g_.m.p = 0
		// 当前m的mcache为nil
		_g_.m.mcache = nil
		// 相应的p的m为0
		_p_.m = 0
		// 设置p的状态为空闲
		_p_.status = _Pidle
		return _p_

	20 runtime.handoffp(_p_ *p)
		// 通过系统调用或者锁定的M，挂起_p_
		// 总是不需要P运行，所以不允许写屏障。
		// handoffp 必须在findrunnable返回G并在_p_上运行的任何情况下启动M。
		
		if !runqempty(_p_) || sched.runqsize != 0 {
						
		}

	21 runtime.runqempty(_p_ *p) bool 
		// runqempty报告_p_在本地运行队列中是否有Gs。
		// 假设它从不返回true
		
		for {
			head := atomic.Load(&_p_.runqhead)
			tail := atomic.Load(&_p_.runqtail)
			runnext := atomic.Loaduintptr((*uintptr)(unsafe.Pointer(&_p_.runnext)))
			// p.runnext，如果非空，是一个由当前G准备的可运行的G。
			// 如果正在运行的G还有时间片，那么接着运行的应该是runnext，而不是
			// 在runq中Gs。它将继承当前时间片剩余的时间。
			// 如果一部分的goroutine被锁定为通信与等待的模式，被设置为一个单元的调度
			// 消除由于将处于准备状态的goroutine添加到队列末尾而产生（可能）很大
			// 调度延迟。

			if tail == atomic.load(&_p_.runqtail) {
				return head == tail && runnext == 0
			}
		}

	22 runtime.startm(_p_ *p, spinning bool)
		// 调度一些M来运行P(如果需要会创建新的M)
		// 如果p为空，尝试获取一个空闲的p，如果没有空闲p怎不做任何处理。
		// 可能运行时m.p为空，所以不允许写屏障。
		// 如果设置了spinning，调用者已经增加nmspinning和startm将减少nmspinning或者在新启动的M中设置m.spinning
		
		lock(&sched.lock)
		if _p_  == nil {
			// 尝试从空闲列表中返回p
			// sched必须被锁定
			// 可能y运行中处于STW，所以不允许写屏障。
			_p_  = pidleget() 
			if _p_ == nil {
				unlock(&sched.lock)
				if spinning {
					
				}
				return
			}
		}

		mp := mget()
		// 尝试从空闲链表中返回m
		unlock(&sched.lock)

		if mp == nil {
			var fn func()
			if spinning {
				fn = mspinning
			}
			newm(fn, _p_)
			return
		}


	23 runtime.newm(fn func(), _p_ *p)	
		// 创建一个新的m，它开始于fn或者调度器的调用
		// fn 需要是静态的并且不是堆申请的闭包
		// 运行的时候m.p可能为nil，所以不允许写屏障

		mp := allocm(_p_, fn)
		mp.nextp.set(_p_)
		mp.sigmask = initSigmask
		if gp := getg(); gp != nil && gp.m != nil && (gp.m.lockedExit != 0 || gp.m.incgo) && GOOS != "plan9" {
			// 我们处于一个上锁的M或者是一个已经启动的C线程。这个线程的内核状态可能很奇怪(用户可能因为处于这个目的锁住它)
			// 我们不希望克隆它到另外的线程中。取而代之的是，要求一个好的的线程为我们创建线程。
			lock(&newmHandoff.lock)
			if newmHandoff.haveTemplateThread == 0 {
				throw("")
			}

			mp.schedlink = newmHandoff.newm
			newmHandoff.newm.set(mp)
			if newmHandoff.waiting {
				newmHandoff.waiting = false
				notewakeup(&newmHandoff.wake)
			}
			unlock(&newmHandoff.lock)
			return
		}
		
		newm1(mp)


	24 runtime.allocm(_p_ *p, fn func()) *m
		// 申请一个和任何线程无关的m
		// 如果需要的话，可以把p用作申请的上下文。
		// fn被记录为新m的m.mstartfn

		// 该函数运行写屏障即使调用者没有这样做，因为它借用了_p_
		//  
		_g_ := getg() //获取当前goroutine
		acquirem() // 禁止GC因为在sysmon中被调用
		if _g_.m.p == 0 {
			// 在函数中为了mallocs借用p
			// 把p和当前m关联起来
			// 
			acquirep(_p_)
		}

		// 释放空闲M列表。我们在某些地方做这个工作同时这可能释放我们可以使用的一个栈。
		if sched.freem != nil {
			lock(&sched.lock)
			for freem := sched.freem; freem != nil; {
				if freem.freewait != 0 {
					next := freem.freelink
					freem.freelink = newList
					newList = freem
					freem = next
					continue
				}
				stackfree(freem.g0.stack)
				freem = freem.freelink
			}
			sched.freem = newList
			unlock(&sched.lock)
		}
		
		mp := new(m)
		mp.mstartfn = fn
		// 对mp 线程进行初始化，主要是对mp.gsignal设置
		mcommoninit(mp)

		if iscgo || GOOS == "solaris" || GOOS == "illumos" || GOOS == "windows" || GOOS == "plan9" || GOOS == "darwin" {
			// 初始化 mp.g0,其g0.stack为空
			mp.g0 = malg(-1)
		} else {
			// 初始化 mp.g0
			mp.g0 = malg(8192 * sys.StackGuardMultiplier)
		}
		mp.g0.m = mp

		if _p_ == _g_.m.p.ptr() {
			releasep()
		}
		releasem(_g_.m)
		
		return mp

	25 runtime.acquirep(_p_ *p)
		// 这部分的执行不允许写屏障
		// wirep是acquirep的第一步，它实际上是将当前M和_p_关联起来。
		// 所以这部分我们不允许写屏障，因为我们没有一个P。
		// 设置_g_.m.p.set(_p_)
		// _p_.m.set(_g_.m)
		wirep(_p_)	
		// 拥有P，现在允许写屏障

		// 在P从可能旧的mcache中申请之前，执行延迟的mcache刷新
		_p_.mcache.prepareForSweep()

	26 runtime (c *mcache)prepareForSweep()
		// 当c已经发布，prepareForSweep会刷新c如果系统已经进入一个新的sweep(扫描)阶段
		// 这肯定发生在sweep阶段开始和第一次申请之间。

		// 另外，为了替换 我们确保每一个P在(starting the world)和分配之间的都这样做，
		// 我们可以开启allocte-black，允许申请像平常一样继续，用一个ragged barrier在
		// 扫描的开始来确保所有缓存的spans被扫描，然后禁用allocate-black。然而，出于
		// 这个目的很难避免位标记蔓延至下个GC循环。

		sg := mheap_.sweepgen
		if c.flushGen == sg {
			return
		} else if c.flushGen != sg-2 {
			throw("bad flush")
		}

		c.releaseAll()
		stackache_clear(c)
		atomic.Store(&c.flushGen, mheap_.sweepgen)

	27 runtime (c *mcache)releaseAll
		for i := range c.alloc {
			s := c.alloc[i]
			if s != &emptymspan {
				// s非空mspan
				mheap_.central[i].mcentral.uncacheSpan(s)
				c.alloc[i] = &emptymspan
			}
		}
		
		
	28 runtime (c *mcentral) uncacheSpan(s *mspan)
		// 从mcache中返回span
		if s.allocCount == 0 {
			throw("uncaching span but s.allocCount == 0")
		}

		sg := mheap_.sweepgen
		// 
		stale := s.sweepgen == sg+1
		if stale {
			// 在标记开始之前，span已经缓存，扫描它是我们的责任。
			// 设置sweepgen来指明它没有缓存但是需要标记，但是不能从
			// 这申请。sweep将设置s.sweepgen来指明s正在扫描
			atomic.Store(&s.sweepgen, sg-1)
		} else{
			// 指明s不再需要标记 
			atomic.Store(&s.sweepgen, sg)
		}

		n := int(s.nelems) - int(s.allocCount)
		if n > 0 {
			// cacheSpan更新alloc，猜想s上面的所有对象都被申请。调整为任何没有
			// 在潜在的标记span之前 我们必须做这个。
			atomic.Xadd64(&c.nmalloc, -int64(n))
			lock(&c.lock)
			c.empty.remove(s)
			c.nonempty.insert(s)
			if !stale {
				// mCentral_CacheSpan 保守的计算heap_live中未分配的槽位，撤销这个
				// 如果标记前span已经缓存，接着heap_live完全是重新计算的，因为缓存了缓存了这个span，所以我们对于陈旧的
				// span作这样的操作。
				// 
				atomic.Xadd64(&memstat.heap_live, -int64(n)*int64(s.elemsize))
			}
			unlock(&c.lock)
		}

		if stale {
			// 现在s正在正确的mcentral列表中，我们可以标记它。
			s.sweep(false)
		}

	29 runtime.mspan
		//go:notinheap
		
		// 链表中下一个mspan，如果没有为nil
		next *mspan
		// 链表中前一个mspan，如果没有为nil
		prev *mspan
		// 用作debug，TODO: Remove
		list *mSpanList

		// span第一个字节的地址，又名s.base()
		startAddr uintptr
		// span中页数目
		npages uintptr 	

		// 在mSpanManual mspan中的空闲对象链表
		manualFreeList gclinkptr

		// freeindex 是从0到nelems之间的索引 在span中用来扫描下一个空闲的对象
		// 每次申请从freeindex开始扫描allocBits直到遇到0，用来指示一个空闲的对象
		// freeindex随后调整所以下次扫描开始于刚刚发现的空闲对象
		
		// 如果freeindex == nelem，这span没有空闲的对象
		
		// allocBits是span中对象的位图，如果n是大于freeindex并且allocBits[n/8] & (1<<(n%8)) == 0
		// 该对象n是空闲的；否则，对象n已经被申请。
		// 开始于nelem的Bits还没有被定义，所以不能被引用。
		// 
		// 对象n开始于地址 n*elemsize + (start << pageShift)
		freeindex uintptr

		// span中对象的数目
		nelems uintptr
		
		// 在freeindex中缓存allocBits，移动allocCache，使得最低位对应index的位。
		// allocCache 持有allocBits的补充，因此允许ctz(count trailing zero)直接使用它。
		// allocCache可能包含超越s.nelems的位；调用者必须忽视它。
		allocCache uint64

		// allocBits和gcmarkBits持有指向span标记和申请位的指针。这些指针是8字节对齐。
		// 由三块持有数据的区域
		// free: 不再访问的Dirty区域可以被重用
		// next: 持有下个GC循环使用到信息
		// current: 当前GC循环中使用的信息
		// previous: 上一个GC循环中使用的信息
		// 一个新的GC循环开始于finishsweep_m的调用。finishsweep_m将previous区域移到free
		// 区域，current区域移到previous区域，next区域移到current区域，
		// next区域将被填充，因为span请求内存为下一个GC循环保存gcmarkBits，并为新分配的
		// span保存allocBits
		
		// 指针运算是靠手工完成，取代数组为了避免在关键路径上进行边界检查。
		// 标记阶段将会释放旧的allocBits，同时设置allocBits为gcmarkBits。gcmarkBits
		// 被替换为一个新的零化内存。
		allocBits *gcBits
		gcmarkBits *gcBits

		// 标记代数
		// 当 sweepgen == h->sweepgen-2, span需要被标记
		// 当 sweepgen == h->sweepgen-1, span正在被标记
		// 当 sweepgen == h->sweepgen, span已经被标记和准备使用
		// 当 sweepgen == h->sweepgen+1, 在开始扫描之前缓存了span，现在仍然缓存，需要进行扫描
		// 当 sweepgen == h->sweepgen+3, span被扫描，接着缓存，仍然需要缓存
		// 每次GC之后，h->sweepgen增加2

		sweepgen uint32
		// 
		divMul uint16
		// 如果非零，elemsize是2的幂，&它会得到对象分配基地址
		baseMask uint16
		// 已经分配的对象数目
		allocCount uint16
		// 类型级别和nosacn
		spanclass spanClass	

		// mspan是基于页面的
		// 当msapn处于heap free treap(空闲树堆)，状态为mSpanFree, 同时heapmap(s->start)等于span;
		// heapmap(s->stars+s->npages-1) == span。
		// 当mspan处于heap scav treap(scav树堆), 那么除了上述scavenged=true，在其他情况下，scavenged=false。
		//
		// 当一个mspan被申请时，状态等于mSpanInUse或者mSpanManual，heapmap(i)等于span 对于所有的s->start <= i < s->start+s->npages
		// 每个mspan都是在双向链表中，不是在mheap的busy链表中，就在其中一个mcentral的span链表中。
		
		// mspan代表着真实内存，拥有这些状态：mSpanInUser，mSpanManual或者mSpanFree。这些状态的变迁限制如下：
		// * span可能从空闲跃迁为in-use或者manual在任何GC阶段
		// * 在标记过程(gcpase == _GCoff)，span可能从in-use跃迁为free(作为标记的结果)，或者manual跃迁为free(作为栈被释放的结果)
		// * 在GC(gcphase != _GCoff), span 不能从in-use或者manual跃迁为free。因为并发GC可能读一个指针，然后查找它的span，span
		// 状态必须是单调不变的。
		// mSpanInUse 支持垃圾回收的堆申请
		// mSpanManual 手动管理的申请
		state mSpanState
		
		// 在申请之前是否需要零化
		needzero uint8
		// 除以elemsize - divMagic.shift
		divShift uint8
		// 除以elemsize - divMagic.shift2
		divShift2 uint8
		// span的页是否释放给回OS
		scavenged bool
		// sizeclass或者npages计算得到
		elemsize uintptr
		// span的结束数据
		limit uintptr
		// 守卫特殊队列
		speciallock mutex
		// 按偏移量排序的特殊记录的链表
		// specials *special

	30 runtime.stackcache_clear(c *mcache)
		lock(&stackpoolmu)	
		for order := uint8(0); order < _NumStackOrders; order++ {
			x := c.stackcache[order].list
			for x.ptr() != nil {
				y := x.ptr().next
				stackpoolfree(x, order)
				x = y
			}
			c.stackcache[order].list = 0
			c.stackcache[order].size = 0
		}
		unlock(&stackpoolmu)
	
	31 runtime.stackpoolfree(x gclinkptr, order uint8)
		// 将栈x加入到空闲pool，调用时必须持有stackpoolmu
		s := spanOfUnchecked(uintptr(x))
		if s.state != mSpanManual {
			// msapn申请区域不是stack，throw
			throw("")
		}
		if s.manualFreeList.ptr() == nil {
			stackpool[order].insert(s)
		}
		x.ptr().next = s.manualFreeList
		s.manualFreeList = x
		s.allocCount--
		
		if gcphase == _GCoff && s.allocCount == 0 {
			// Span是完全空闲，如果我们正在标记的话，立刻返回给heap。
			// 如果GC是活跃，我们推迟释放直到GC结尾，为了避免如下类型的场景
			// 1) GC开始，扫描一个SudoG但是还没有标记SudoG.elem指针
			// 2) 指向栈的指针被复制
			// 3) 旧的栈被释放
			// 4) 包含的span被标记为空闲
			// 5) GC尝试标记SudoG.elem指针。标记失败因为指针看起来在空闲span里面。
			// 
			stackpool[order].remove(s)
			s.manualFreeList = 0
			osStackFree(s)
			mheap_.freeManual(s, &memstat.stacks_inuse)
		}
	

	32 runtime (m *mheap) freeManual(s *mspan, stat *uint64)
		// freeManual 释放一个由allocManual申请的手工管理的span，stat必须等于在申请s时传给allocManual的stat。
		// 这只能当gcphase=_GCoff被调用。
		// freeManual必须在系统栈中被调用因为它需要heap锁。

		s.needzero = 1
		lock(&h.lock)
		*stat -= uint64(s.npages << _PageShift)
		memstat.heap_sys += uint64(s.npages << _PageShift)
		h.freeSpanLocked(s, false, true)
		unlock(&h.lock)
	
	33 runtime (h *mheap) freeSpanLocked(s *mspan, acctinuse, acctidle bool)
		switch s.state {
		case mSpanManual:
			if s.allocCount != 0 {
				throw("")
			}
		case mSpanInUse:
			if s.allocCount != 0 || s.sweepgen != h.sweepgen {
				throw("")
			}
			// mheap减去mspan page
			h.pageInUse -= uint64(s.npages)
			
			arena, pageIdx, pageMask := pageIndexOf(s.base())
			arena.pageInuse[pageIdx] &^= pageMask
		default:
			throw("")
		}

		if acctinuse {
			memstat.heap_inuse -= uint64(s.npages << _PageShift)
		}
		if acctidle {
			memstat.heap_idle += uint64(s.npages << _PageShift)
		}
		s.state = mSpanFree

		h.coalesce(s)

		h.free.insert(s)

	34 runtime pageIndexOf(p uintptr) (arena *heapArena, pageIdx uintptr pageMask uint8)
		ai := arenaIndex(p)
		arena = mheap_.arenas[ai.l1()][ai.l2()]
		pageIdx = ((p / pageSize) / 8) % uintptr(len(arena.pageInUse))
		pageMask = byte(1 << ((p / pageSize) % 8))


	35 runtime.spanOfUnchecked(p uintptr) *mspan 
		// 先通过指针p找出p所属的arena index，然后再通过mheap_.arenas的映射关系找到响应的mspan
		//go:nosplit
		// spanOfUnchecked 等价于 spanOf，不过调用者必须确保指针p处于可申请的堆区域中。
		ai := arenaIndex(p)
		return mheap_.arens[ai.l1()][ai.l2()].spans[(p/pageSize)%pagesPerArena]

	36 runtime.spanOf(p uintptr) *mspan
		// spanOf返回p所属的span。如果p不是指向heap arena或者没有span包含指针p，spanOf返回nil
		// 如果p不是指向已经申请的内存，有可能返回不包含p的non-nil的span。如果可能，调用者应该
		// 调用spanOfHeap或者明确地检查span边界。
		// 必须是nosplit因为调用者是nosplit

		// 这个函数看起来大，但是我们在arenaL1Bits上面使用大量的常量用来将它控制在预算之内。另外
		// 这里很多检查都是Go不论如何都是需要作的安全检查，所以生产代码非常短。
		ri := arenaIndex(p)
		if arenaL1Bits == 0 {
			if ri.l2() >= uint(len(meahp_.arenas[0])) {
				return nil
			}
		} else {
			if ri.l1() >= uint(len(mheap_.arenas)) {
				return nil
			}
		}
		
		l2 := mheap_.arenas[ri.l1()]
		if arenaL1Bits != 0 && l2 == nil {
			return nil
		}

		ha := l2[ri.l2()]
		if ha == nil {
			return nil
		}

		return ha.spans[(p/pagesize)%pagesPerArena]
	

	37 runtime.mheap
		//go:noinheap
		
		// heap本身就是"free"和"scav"树堆，但是其他的全局数据也在这里。
		// mheap不能是堆分配的因为它包含了不能堆分配的mSpanLists。

		// 锁只能在系统栈获取，否则g可能死锁如果栈扩展的时候持有锁。
		lock mutex
		// 空闲span
		free mTreap
		// 标记generation
		sweepgen uint32
		// 所有span被标记
		sweepdone uint32
		// 活跃sweepone调用次数
		sweepers uint32
		
		// allspans是所有创建过的mspans切片。每个mspan只出现一次。
		// allspans的内存手工管理，可以重新申请和随着heap增长移动
		// 
		// 通常，allspans由mheap_.lock保护，阻止并发访问和备份存储的释放
		// 在STW期间访问可能没有持有锁，但必须确保在访问期间不能发生申请
		allspans []*mspan

		// sweepSpans维护两个mspan栈，一个是标记在使用的mspan，另外一个
		// 未标记在使用的msapn。在每次GC期间有2种主要的角色。由于在每次GC循环中，
		// sweepgen增加2，这就意味标记的spans在sweepSpans[sweep/2%2]和未标记的
		// spans在sweepSpans[1-sweepgen/2%2]。标记从未标记的stack pop出来的span,
		// push 仍然in-use的spans入栈。同样地，分配一个正在使用的span会push到
		// 以标记的栈
		sweepSpans [2]gcSweepBuf

		// 这些参数表示从heap_live到页面扫描计数的一个线性函数。均衡的sweep
		// 系统给标记为黑色在当前heap_live通过将当前扫描计数保持在该线上面


		// 这条线的斜率为sweepPagesPerByte，通过一个基点(sweepHeapLiveBasis，pagesSweptBasis)
		// 在任何时间，系统位于(memstats.heap_live，pagesSwept)在这片空间。
		// 重要的是，这条直线穿过我们控制的一个点，而不是简单地从原点(0,0)开始，因为这样
		// 可以让我们在考虑当前进程的同时调整扫描速率。如果我们只是调整斜率，它会产生不连续
		// 债务如果有任何进展的话。

		// 统计中的spans页
		pagesInUse uint64
		// 这个循环中标记页面，原子更新
		pagesSwept uint64
		// pagesSwept 用来作为原始标记比率，原子更新
		pagesSweptBasis uint64
		// heap_live中的值用来作为原始标记比率
		sweepHeapLiveBasis uint64
		// 均衡的标记比率
		sweepPagesPerByte float64

		
		// 回收速率参数
		// 2个基础参数和回收比率平衡均衡的标记实现，基本区别是:
		// *回收关注RSS，估计为heapRetained()
		// *不是推送回收到GC，它被定步到一个基于时间的速率计算在gcPaceScavenger

		// scavengeRetainedGoal代表这我们的目标RSS
		// 所有的字段访问必须通过锁
		// 
		scavengeTimeBasis int64
		scavengeRetainedBasis uint64
		scavengeBytesPerNS float64
		scavengeRetainedGoal uint64
		scavengeGen uint64

		// 页回收状态
		// recaimIndex是在下一页allArens的页面索引用来回收，特别是
		// 它指向arena allArenas[i / pagePreArena]的page(i%pagesPerArena)
		// 如果它大于等于 1<<63，页面reclaimer已经做了扫描页面标记
		reclaimIndex uint64

		// reclaimCredit是用于额外页面扫描的备用信贷。由于页面reclaimer工作
		// 在大量的块中，他可能相比于请求回收更多。任何备用页面释放会回到
		// 信贷pool
		reclaimCredit uintptr
		
		// Malloc统计
		// 用于大对象申请的字节数
		largealloc uint64
		// 大对象申请的数目
		nlargealloc uint64
		// 用于大对象的空闲字节数 >maxsmallsize
		largefree uint64
		// 空闲大对象数目
		nlargefree uint64
		// 空闲小对象数目
		nsmallfree [_NumSizeClassed]uint64

		// arenas是堆arena map。它指向整个可用虚拟地址空间的每个arena帧的堆元数据
		// 使用arenaIndex来计算在这个切片的index
		// 对于Go heap不支持的地址空间区域，arena map可能包含nil
		// 修改被mheap_.lock保护，执行读不需要锁；然而，一个给定
		// 实例在没有持有锁的情况下可以从nil过渡到non-nil。(实例从不
		// 过渡回来为nil)
		// 通常，这是一个2层映射由L1map和可能许多L2maps组成。当他们是大量arena栈时，这样节省空间。
		// 然而，在很多平台(即使是64-bit)，arenaL1Bits等于0，使这成为
		// 一个有效的简单等级的map。在这种场景下，arena[0]永远不会为nil
		arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

		// heapArenaAlloc 是一个块用于申请heapArena对象的预保留。
		// 这只使用在32-bit上，我们预保留这块空间避免和堆本身交叉。
		heapArenAlloc linearAlloc

		// arenaHist是用来添加更多heap arenas的地址列表。它最初由一组通常提示地址填充，伴随这实际堆范围边界增长。
		arenaHints *arenaHint

		// arena 是一块用于申请heap arenas(真实arenas)的预保留空间，只是用在32-bits上。
		arena linearAlloc
		
		// allArenas 是每个映射arena的arenaIndex。它可以用来遍历地址空间
		// 访问由mheap_.lock保护，然而，由于只是追加和老的备份数组从不释放，获取mheap_.lock是安全的，复制切片头，然后释放mheap_.lock。
		allArenas []arenaIdx
	
		// sweepArenas 是发生在标记循环开始的allArenas快照。通过阻塞GC(或者机制抢占)可以安全地读。
		sweepArenas []arenaIdx
		
		// curArena 是heap正在增长的arena。它应该总是物理页对齐的。
		curArena struct {
			base, end uintptr
		}

		// central 空闲列表面对小类型对象
		// padding 确保mcentrals是以CacheLinePadSize字节分隔，所以每个mcentral.lock得到自己的cache line
		// central的索引是spanClass
		central [numSpanClass]struct {
			mcentral mcentral
			pad [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
		}

		// span申请者
		spanalloc fixalloc
		// mcache申请者
		cachealloc fixalloc
		// treapNodes申请者
		treapalloc fixalloc
		// specialfinalizer申请者
		specialfinalizeralloc fixalloc
		// specialprofile申请者
		specialprofilealloc
		// special record 申请者的锁
		speciallock mutex
		// arenaHints 申请者
		arenaHintAlloc fixalloc
		
		// 从不设置，这里只是强制specailfinalizer类型为DWARF
		unused *specialfinalizer

	38 runtime (h *mheap) coalesce(s *mspan)
		// 合并相邻mspan

		// merge 是一个帮助器用来将其他合并到s，删除heap元数据中对这些的引用，然后丢弃他们。这些mspan必须与s相邻
		merge := func(a, b, other *mspan) {
			// 调用者必须确保a.startAddr < b.startAddr 和a或者b是s。a和b必须是相邻的。other是两者中不是s的一个。
			if pageSize < physPageSize && a.scavenged && b.scavenged {
				// 如果我们在pageSize < physPageSize的系统上合并2个回收的spans，他们的边界总是在物理页边界上，因为重组发生在合并过程。
				_, start := a.physPageBounds()
				end, _ := b.physPageBounds()
				if start != end {
					throw("")
				}
			}
	
			// 通过base和npages调整s，同时也是在heap 元数据中
			s.npages += other.npages
			s.needzero |= other.needzero
			if a == s {
				// s.base()+s.npages*pageSize-1是合并后mspan的尾地址
				h.setSpan(s.base()+s.npages*pageSize-1, s)
			} else {
				s.startAddr = other.startAddr
				h.setSpan(s.base(), s)
			}
	
			// 大小可能正在改变，所以treap需要删除相邻的对象，同时作为一个联合节点添加回去。
			// 从mheap_.free treap找到mspan所属的treap节点，并移除。 
			h.free.removeSpan(other)
			// 设置该mspan的状态为 mSpanDead
			other.state = mSpanDead
			h.spanalloc.free(unsafe.Pointer(other))
		}
	
		// realign 是一个帮助器用于收缩other和扩容s使得他们的边界在一个物理页边界上。	
		realign := func(a, b, other *mspan) {
			// 调用者必须确保a.startAddr < b.startAddr同时a和b有它一个是s，a和b必须相邻。other是2者之中不是s那个。
			// 如果pageSize >= physPageSize所以spans总是对齐物理页边界，所以只是退出。
			if pageSize >= physPageSize {
				return
			}
			// 由于我们重新调整other，我们必须从treap移除它
			h.free.removeSpan(other)
			
			// Round的边界接近最接近物理页大小，接近已回收的span
			boundary := b.startAddr
			if a.scavenged {
				boundary &^ = (physPageSize-1)
			} else {
				boundary = (boundary + physPageSize-1) &^ (physPageSize -1)
			}
			a.npages = (boundary - a.startAddr)/pageSize
			b.npages = (b.startAddr + b.npages*pageSize-boundary) / pageSize
			b.startAddr = boundary

			h.setSpan(boundary-1, a)
			h.setSpan(boundary, b)

			h.free.insert(other)
		}

		hpMiddle := s.hugePages()
		
		// Coalesce with earlier, latesr spans
		var hpBefore uintptr
		// 获取s前个mpsan: before，检查before的状态
		if before := spanOf(s.base()-1); before != nil && before.state == mSpanFree {
			// 如果s的回收状态==before的回收状态，则合并
			if s.scavenged == before.scavenged {
				hpBefore = before.hugePages()
				merge(before, s, before)
			} else {
			// 重新调整(重新调整s和befefore的边界)
				realign(before, s, before)
			}	
		}

		// 现在检查下个span(地址更大)是否空闲和是否可以合并
		var hpAfter uintptr
		if after := spanOf(s.base()+s.npages*pageSize); after != nil && after.state == mSpanFree {
			// s.base+s.npages*pageSize 是属于下个mspan的
			if s.scavenged == after.scavenged {
				hpAfter = after.hugePages()
				merge(s, after, after)
			} else {
				realign(s, after, after)
			}
		}

		// 如果s还没有将page归还给os并且 hugePages数量更多
		if !s.scavenged && s.hugePages() > hpBefore+hpMiddle+hpAfter {
			// 如果s已经增长以致它可能包含更多的huge page相比它和它现在合并的邻近mspan，则标记整个地区作为huge-page-backable
			// 另外，在我们打碎huge pages(比如linux)的系统上，huge pages可能不支持s，因为它可能由许多底层的VMA组成。主要问题是
			// 它可能导致对由大页面支持的空闲内存数量的估计不足，从而无法确定清楚速率。
			
			// 
			sysHugePage(unsafe.Pointer(s.base(), s.npages*pageSize))
		}


		
	39 runtime (s *mSpan) hugePages() uintptr 
		// physPageSize 是操作系统物理页字节大小，Mapping和unmapping操作都是
		// 在多重物理页上完成的。
		// physHugePageSize 是操作系统默认物理大页的字节大小，它的申请对于应用不是透明的
		// 它被猜测和验证是2的幂
		
		// 如果设置，这必须是由操作系统初始化代码(典型在osinit中)设置，在mallocinit字前。然而
		// 设置它是可选的，同时保留默认值总是安全的(尽管可能效率较低)

		// 由于physHugePageSize总是被猜测是2的幂，physHugePageSize 被定义为physHugePageSize == 
		// 1 << physHugePageShift。physHugePageShift的目的是用来避免在性能关键函数上作划分。

		// hugePages 返回mspan所拥有的内存区域中的对齐物理页数量
		if physHugePageSize == 0 || s.npages < physHugePageSize/pageSize {
			return 0
		}
		start := s.base()
		end := start + s.npages*PageSize
		if physHugePageSize > pageSize {
			// start对齐physHugePageSize
			start = (start + physHugePageSize -1) &^ (physHugePageSize -1)
			end &^ = physHugePageSize -1
		}
		if start < end {
			return (end -start) >> physHugePageShift
		}
		return 0


	40 runtime stackfree(stk stack)
		// go:systemstack
		// stackfree释放stk上的n个字节stack申请。
		// stackfree必须运行在系统栈上因为它用P的资源和必须不能扩张stack

		// 获取当前goroutine
		gp := getg()
		v := unsafe.Pointer(stk.lo)
		// stack的大小
		n := stk.hi - stk.lo
		if n&(n-1) != 0 {
			// 是否是2的幂
			throw("")
		}
		if stk.lo+n < stk.hi {
			throw("")
		}
		fi stackDebug >= 1 {
			println("stackfree", v, n)
			memclrNoHeapPointers(v, n)
		}
		if debug.efence != 0 || stackFromSystem != 0 {
			if debug.efence != 0 || stackFaultOnFree != 0 {
				sysFault(v, n)
			} else {
				sysFree(v, n, &memstats.stacks_sys)
			}
			return
		}
		if msanenables {
			msanfree(v, n)
		}
		
		if n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {
			order := uint8(0)
			n2 := n
			for n2 > _FixedStack {
				order++
				n2 >> 1
			}
			x := gclinkptr(v)
			c := gp.m.mcache
			if stackNoCache != 0 || c == nil || gp.m.preemptoff != "" {
				lock(&stackpoolmu)
				// 该stack是在heap内申请
				stackpoolfree(x, order)
				unlock(&stackpoolmu)
			} else {
				if c.stackcache[order].size >= _StackCacheSize {
					// 该stack是在heap内申请
					stackcacherelease(c, order)
				}
				x.ptr().next = c.stackcache[order].list
				c.stackcache[order].list = x
				c.stackcache[order].size += n
			}
		} else {
			// 根据地址uintptr(v)获取该地址所属的mspan:s
			s := spanOfUnchecked(uintptr(v))
			// 如果mspan状态不是mSpanManual，则抛出
			if s.state != mSpanManual {
				throw()
			}
			if gcphase == _GCoff {
				// 如果我们正在sweeping，立即释放stack
				osStackFree(s)
				// 手动释放mspan:s
				mheap_.freeManual(s, &memstats.stacks_inuse)
			} else {
				// 如果GC正在运行，我们不能将stack span返回给heap 因为它可能被当作a heap span重新使用。
				// 同时它的状态改变会和GC发生竞争。取而代之的是将它添加到large stack cache。
				log2npage := stacklog2(s.npages)
				lock(&stackLarge.lock)
				stackLarge.free[log2npage].insert(s)
				unlock(&stackLarge.lock)
			}	
		}

	41 runtime stackcacherelease(c *mcache, order uint8)
		//go:systemstack

		if stackDebug >= 1 {
			print("")
		}
		x := c.stackcache[order].list
		size := c.stackcache[order].size
		lock(&stackpoolmu)
		for size > _StackCacheSize/2 {
			y := x.ptr().next
			stackpoolfree(x, order)	
			x = y
			size -= _FixedStack << order
		}
		unlock(&stackpoolmu)
		c.stackcache[order].list = x
		c.stackcache[order].size = size

	42 runtime.heapArena struct 
		// heapArena存储heap arena的数据单元。heapArenas储存在Go heap
		// 之外，可以通过mheap_.arenas 索引访问。
		// 它直接通过OS申请，所以理想情况下它应该是系统页的倍数。举例，避免添加小字段。
		
		// bitmap 存储在这个区域的words的指针/标量bitmap，适合heapBits类型可以访问它
		bitmap [heapArenaBitmapBytes]byte
		// spans映射这片区域的虚拟页ID到*mspan，对于申请的spans，它们的页映射span本身。
		// 对于空闲的span，只有最低位和最高位的页映射span本身。内部页映射到任意span。
		// 对于从来没有被申请的页面，spans入口为空 
		// 
		// 修改通过mheap.lock保护。不需要锁可以执行读操作，但是只能执行已知包含在使用或者
		// 栈的span的索引。这就意味着在确定这个地址是会否有效和在spans数组中查找之间肯定不是
		// 安全点。
		spans [pagePerArena]*mspan

		// pageInUse是一个指定那个spans处于mSpanInUse状态的bitmap。这个bitmap的索引是页号
		// 但只是bit对应到每个使用的span的第一页。
		pageInUse [pagePerArena/8]uint8

		// pageMarks 是一个指明那个spans拥有标记对象的bitmap，类似pageInUse,只是每个使用的
		// span的第一页与bit对应。

		// 在标记过程中，写是原子完成。读不是原子性和锁空闲的，因为他们只是在扫描期间发生。(因此和读从不竞争)
		// 这只是用来快速查找这整个span可以释放。
		// 
		pageMarks [pagePerArena/8]uint8
	
	43 runtime mcommoninit(mp *m)
		// m初始化
		_g_ := getg()

		// g0 stack对于用户来说没有意义
		if _g_ != _g_.m.g0 {
			callers(1, mp.createstack[:])
		}
		
		lock(&sched.lock)
		if sched.mnext+1 < sched.mnext {
			throw("")
		}
		mp.id = sched.mnext
		sched.mnext++
		// 检查sched.maxmcount(msched允许的最大值)
		checkmcount()

		mp.fastrand[0] = 1597334677 * uint32(mp.id)
		mp.fastrand[1] = uint32(cputicks)
		// 检查mp.fastrand[0]和mp.fastrand[1]是否都为0
		if mp.fastrand[0]|mp.fastrand[1] == 0 {
			mp.fastrand[1] = 1
		}

		// 初始化mp的gsignal
		mpreinit(mp)
		if mp.gsignal != nil {
			mp.gsignal.stackguard1 = mp.gsignal.stack.lo + _StackGuard
		}

		// 添加allm所以garbage collector不会释放g->m当它只是在寄存器或者线程本地存储
		mp.alllink = allm

		atomicstorep(unsafe.Pointer(&allm), unsafe.Pointer(mp))
		unlock(&sched.lock)

		if iscgo || GOOS == "solaris" || GOOS == "illmos" || GOOS == "windows" {
			mp.cgoCallers = new(cgoCallers)
		}
	

	44 runtime mpreinit(mp *m)
		// 调用它来初始化一个新创建的m(包括bootstrap m)
		// 在父协程中调用它(main thread在bootstrap情况下),可能申请内存

		// mp.gsignal = 新申请的协程
		mp.gsignal = malg(32 * 1024)
		// gsignal的m = 原来的线程
		mp.gsignal.m = mp
	
	45 runtime malg(stacksize int32) *g
		// 申请一个满足stacksize字节的栈的goroutine
		
		newg := new(g)
		if stacksize >= 0 {
			stacksize = round2(_StackSystem+stacksize)
			systemstack(func() {
				newg.stack = stackalloc(uint32(stacksize))	
			})
			newg.stackguard0 = newg.stack.lo + _StackGuard
			newg.stackguard1 = ^uintptr(0)
		}
			
		return newg
	
	46 runtime stackalloc(n uint32) stack 
		//go:systemstack

		// stackalloc 申请一个n字节stack
		// stackalloc必须运行在系统栈因为它用per-P的资源同时必须不能分割栈。
		// stackalloc必须在调用栈中执行，因此我们从不尝试扩展这个栈在代码stackalloc运行的期间
		// 如果会导致死锁

		thisg := getg()		
		if thisg != this.m.g0 {
			//不是运行在调度栈上
			throw("")
		}

		if n&(n-1) != 0 {
			// 栈的大小n 不是2的幂
			thow("")
		}
		
		if stackDebug >= 1 {
			print("")
		}

		if debug.efence != 0 || stackFromSystem != 0 {
			// n对齐physPageSize
			n = uint32(round(uintptr(n), physPageSize))	
			// 从系统申请内存地址，v是这块地址开始地址
			v := sysAlloc(uintptr(n), &memstats.stacks_sys)
			if v == nil {
				throw("")
			}
			return stack{uintptr(v), uintptr(v)+uintptr(n)}
		}

		var v unsafe.Pointer
		if n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {
			// 小stack通过一个固定大小，空闲列表的申请器申请。如果我们需要一个更大的stack，我们
			// 依赖申请一个专用的span。
			
			order := uint8(0)
			n2 := n
			if n2 > _FixedStack {
				order++
				n2 >>= 1
			}
			var x gclinkptr
			c := thisg.m.mcache
			if stackNoCache != 0 || c == nil || thisg.m.preemptoff != "" {
				// c == nil 可能发生在内部的退出系统调用或者procresize。只是从全局pool中
				// 获取一个栈。另外在GC期间不能访问stackcache，因此是并发刷新。	
				lock(&stackpoolmu)
				x = stackpoolalloc(order)
				unlock(&stackpoolmu)
			} else {
				x = c.stackcache[order].list
				if x.ptr() == nil {
					stackcacherefill(c, order)
					x = c.stackcache[order].list
				}
				c.stackcache[order].list = x.ptr().next
				c.stackcache[order].size -= uintptr(n)
			}

			v = unsafe.Pointer(x)
		} else {
			var s *mspan
			npage := uintptr(n) >> _PageShift
			log2npage := stacklog2(npage)

			// 尝试从large stack cache中获取一个stack
			lock(&stackLarge.lock)
			if !stackLarge.free[log2npage].isEmpty() {
				// 判断stackLarge 空闲mspanList是否为空
				s = stackLarge.Free[log2npage].first
				stackLarge.free[log2npage].remove(s)
			}
			unlock(&stackLarge.lock)

			if s == nil {
				// 从heap申请一个新的stack
				s = mheap_.allocManual(npage, &memstats.stacks_inuse)
				if s == nil {
					throw("")
				}
				osStackAlloc(s)
				s.elemsize = uintptr(n)
			}
			v = unsafe.Pointer(s.base())
		}

		if raceenabled {
			racemalloc(v, uintptr(n))
		}
		
		if msanenabled {
			msanmalloc(v, uintptr(n))
		}

		if stackDebug >= 1 {
			print("")
		}

		return stack{uintptr(v), uintptr(v)+uintptr(n)}

	47 runtime sysAlloc(n uintptr, sysStat *uint64) unsafe.Pointer
		p, err := mmap(nil, n, _PROT_READ|_PROT_WRITE, _MAP_ANON|_MAP_PRIVATE, -1, 0)		
		if err != nil {
			if err == _EACCES {
				print("")
				exit(2)
			}
			if err == _EAGAIN {
				print("")
				exit(2)
			}
			return nil	
		}
		mSysStatInc(sysStat, n)
		return p

	48 runtime stackpoolalloc(order uint8) gclinkptr
		// 从空闲池中申请一个stack，调用时必须持有stackpoolmu
		list := &stackpool[order]
		s := list.first
		if s == nil {
			// 无空闲stacks. 申请另一个有效的span
			s = mheap_.allocManual(_StackCacheSize>>_PageShift, &memstat.stacks_inuse)
			if s == nil {
				throw("")
			}
			if s.allocCount != 0 {
				throw("")
			}
			if s.manualFreeList.ptr() != nil {
				throw("")
			}

			// osStackAlloc执行操作系统指定的初始化工作在s作为stack 内存使用之前
			osStackAlloc(s)
			// 元素类型大小为 _FixedStack<<order
			s.elemsize = _FixedStack << order

			for i := uintptr(0); i < _StackCacheSize; i += s.elemsize {
				// 构建空闲链表
				x := gclinkptr(s.base()+i)
				x.ptr().next = s.manualFreeList
				s.manualFreeList = x
			} 
			// 插入到相应的mSpanList
			list.insert(s)
		}

		x := s.manualFreeList
		if x.ptr() == nil {
			throw("")
		}

		s.manualFreeList = x.ptr().next
		s.allocCount++
		if s.manualFreeList.ptr() == nil {
			// mspan中的所有stacks已被申请
			list.remove(s)
		}
		return x

	49 runtime (h *mheap) allocManual(npage uintptr, stat *uint64) *mspan
			// allocManual 申请一个手动管理的由npage页组成的span，如果申请失败allocManual返回nil
			// allocManual添加使用的字节到*stat，这是memstats in-use字段。不像在GC heap的申请，
			// 这个申请不会计入heap_inuse或者heap_sys
			// 如果span.needzero设置，返回的span中的memory可能不是零。
			// allocManual 必须在系统栈中调用因此它需要heap锁。
			lock(&h.lock) 
			// 从mheap_.free堆树申请
			s := h.allocSpanLocked(npage, stat) 
i			if s != nil {
				s.state = mSpanManual
				s.manualFreeList = 0
				s.allocCount = 0
				s.spanclass = 0
				s.nelems = 0
				s.limit = s.base() + s.npages<<_PageShift
				// 手动管理的内存不计入heap_sys
				memstats.heap_sys -= uint64(s.npages << _PageShift)
			}

			unlock(&h.lock)
			return s
	
	50 runtime (h *mheap) allocSpanLocked(npage uintptr, stat *uint64) *mspan
			// 申请一个给定大小的span。h必须被锁
			// 返回的span已经在空闲结构中移除，但是它的状态依然是mSpanFree
			t := h.free.find(npage)	
			// 
			if t.valid() {
				goto HaveSpan
			}

			// 在mheap.free列表中添加npages的mspan
			if !h.grow(npage) {
				return nil
			}
			// 再次寻找
			t = h.free.find(npage)
			if t.valid() {
				goto HaveSpan
			}
			throw("grew heap, but no adequate free span found")

		HaveSpan:
			// 获取mspan
			s := t.span()
			// 如果mspan的状态不为mspanFree，则throw
			if s.state != mSpanFree {
				// 用于申请的候选mspan不是空闲状态
				throw("")
			}
			
			// 首先，从s中减去已经释放给回操作系统的内存。如果需要我们将会加回剩下的
			// memstats.heap_released减去mspan释放的内存？
			memstats.heap_released -= uint64(s.released())	

			if s.npages == npage {
				// mheap_.free 剔除 mspan:t
				h.free.erase(t)
			} else if s.npages >= npage {
				//
				n := (*mspan)(h.spanalloc.alloc())
				h.free.mutate(t, func(s *mspan) {
					n.init(s.base(), npage)
					s.npages -= npage
					s.startAddr = s.base()+npage*pageSize
					h.setSpan(s.base()-1, n)
					h.setSpan(s.base, s)
					h.setSpan(n.base, n)
					n.needzero = s.needzero
					n.scavenged = s.scavenged
					if s.scavenged {
						start, end := s.physPageBounds()
						if start < end {
							memstats.heap_released += uint64(end-start)
						} else {
							s.scavenged = false
						}
					}
				})
				s = n
			} else {
				// too small
				throw()
			}

			if s.scavenged {
				// sysUsed 在span中有效的所有页。注意我们不需要heap_released减去相应量因为我们之前就已经做了。
				sysUsed(unsafe.Pointer(s.base()), s.npages<<_PageShift)
				s.scavenged = false
			}

			// 范围设置s.base
			h.setSpans(s.base(), npage, s)

			*stat += uint64(npage << _PageShift)
			memstats.heap_idle -= uint64(npage<<_PageShift)

			if s.inList() {
				// in list
				throw("")
			}

			return s

	51 runtime (h *mheap) grow(npage uintptr) bool
			// 尝试添加最少npage页内存到heap，返回是否成功
			ask := npage << _PageShift
			// 对齐物理页
			nBase := round(h.curArena.base+ask, physPageSize)
			if nBase > h.curArena.end {
				// 在当前的arena没有足够的空间。申请更多的arena空间。
				// 这可能会让当前arena不连续，所以我们必须请求完整ask
			
				av, size := h.sysAlloc(ask)
				if av == nil {
					print("")
					return false
				}	
			
				if uintptr(av) == h.curArena.end {
					// 新的地址空间和旧的地址空间是连续的，所以只是扩展当前空间。	
					h.curArena.end = uintptr(av) + asize
				} else {
					// 新的地址空间不连续。跟踪当前保留的空间和切换到新的空间，这应该比较罕见。
					if size := h.curArena.end - h.curArena.bash; size != 0 {
						h.growAddSpan(unsafe.Pointer(h.curArena.bash), size)
					}
					// 切换到新的空间
					h.curArena.base = uintptr(av)
					h.curArena.end = uintptr(av) + asize
				}

				// 刚刚申请的内存计算在被释放和空闲，尽管还没有spans支持。
				// 内存申请总是对齐大于physPageSize的heap arena 大小。 
				// 所以直接添加到heap_released是安全的。合并，如果可能，在统计上总是正确的，因为s.base()必须
				// 一个物理页边界。
				memstats.heap_released += uint64(asize)
				memstats.heap_idle += uint64(asize)
	
				// 重新计算nBase
				nBase = round(h.curArena.bask+ask, physPageSize)
			}

			// 在当前arena增长
			v := h.curArena.base
			h.curArena.base = nBase
			h.growAddSpan(unsafe.Pointer(v), nBase-v)
			return true

	52 runtime (h *mheap) sysAlloc(n uintptr) (v unsafe.Pointer, size uintptr)
			// sysAlloc申请最少n字节的heap arena空间。返回的指针总是heapArenaBytes-aligned对齐和
			// 依靠h.arenas metadata。返回的大小总是heapAreanaBytes多倍。sysAlloc失败时返回nil。
			// 这没有相应的释放函数
		
			// sysAlloc返回处于Prepared状态的内存区域。该区域必须转换转换成Ready状态在使用之前。
			// h必须上锁。
			n = round(n, heapAreanaBytes)		

			// 首先，尝试arena预先判断空间是否足够，不足则通过mmap申请
			v = h.arena.alloc(n, heapAreanaBytes, &memstats.heap_sys)
			if v != nil {
				size = n
				goto mapped
			}
	
			// 尝试在hint地址上扩张heap
			for h.arenaHints != nil {
				hint := h.arenaHints	
				p := hint.addr
				if hint.down {
					p -= n
				}
				if p+n < p {
					v = nil
				} else if arenaIndex(p+n-1) >= 1<<arenaBits {
					// 超过可访问的地址heap
					v = nil
				} else {
					// 系统预留
					v = sysReserve(unsafe.Pointer(p), n)
				}
				if p == uintptr(v) {
					// 成功，更新hint
					if !hint.down {
						p+=n 
					}
					hint.addr = p
					size = n
					break
				}
				// 失败。丢弃这个hint和尝试下一个hint
				
				if v != nil {
					// 预释放
					sysFree(v, n, nil)
				}
				// 指向下一个arenaHints
				h.arenaHints = hint.next
				h.arenaHintAlloc.free(unsafe.Pointer(hint))
			}

			if size == 0 {
				if raceenabled {
					// Race探测器猜测 heap地址范围[0x00c000000000，0x00e000000000]，但是我们
					// 用完这个地区的hints。抛出错误
					throw("")
				}

				// 所有的hints失败，所以我们获取内核分配给我们的的任何对齐地址
				v, size = sysReserveAligned(nil, n, heapArenaBytes)
				if v == nil {
					return nil, 0
				}

				// 为扩展的区域创建新的hints
				hint := (*arenaHint) (h.arenaHintAlloc.alloc())
				hint.addr, hint.down = uintptr(v), true
				hint.next, mheap_arenaHints = mheap_.arenaHints, hint
				hint = (*arenaHint)(h.arenaHintAlloc.alloc())
				hint.addr = uintptr(v) + size
				hint.next, mheap_arenaHints = mheap_.arenaHints, hint
			}

			// 检查坏指针或者我们不能用的指针
			{
				var bad string
				p := uintptr(v)
				if p+size < - {
					throw()
				} else if arenaIndex(p) >= 1<<arenaBits {
					throw()
				} else if arenaIndex(p+size-1) >= 1<<arenaBits {
					throw()
				}
			}

			// 如果v和heapArenaBytes没有对齐
			if uintptr(v) & (heapArenaBytes-1) != 0 {
				throw()
			}
		
			// 将预保留转化为准备
			sysMap(v, size, &memstats.heap_sys)

		mapped:
			// 创建 arena metadata
			// 设置每个>=ri且<ri+size-1
			for ri := arenaIndex(uintptr(v)); ri <= arenaIdex(uintptr(v)+size-1); ri++ {
				l2 := h.arenas[ri.l1()]
				if l2 == nil {
					// 申请一个L2 arena map
					l2 = (*[1 << arenaL2Bits]*heapArena)(presistentalloc(unsafe.Sizeof(*l2), sys.PtrSize, nil))
					if l2 == nil {
						throw("")
					}
					// 设置
					atomic.StorepNoWB(unsafe.Pointer(&h.arenas[ri.l1()]), unsafe.Pointer(l2))
				}

				if l2[ri.l2()] != nil {
					throw("")
				}

				var r *heapArena
				r = (*heapArena)(h.heapArenaAlloc.alloc(unsafe.Sizeof(*r)), sys.PtrSize, &memstats.gc_sys))
				if r == nil {
					r = (*heapArena)(persistentalloc)(unsafe.Sizeof(*r), sys.PtrSize, &memstats.gc_sys)
					if r == nil {
						throw("")
					}
				}

				// 添加arena到列表
				if len(h.allArenas) == cap(h.allArenas) {
					size := 2 * uintptr(cap(h.allarenas)) * sys.PtrSize
					if size == 0 {
						sys = physPageSize
					}

					newArray := (*notInHeap)(persistentalloc)(size, sys.PtrSize, &memstats.gc_sys)
					if newArray == nil {
						throw("")
					}
					oldSlice := h.allArenas
					*(*notInHeapSlice)(unsafe.Pointer(&h.allArenas)) == notInHeapSlice{newArray, len(h.allArenas), int(sys/sys.PtrSize)}
					copy(h.allArenas, oldSlice)
				}
				h.allArenas = h.allArenas[:len(h.allArenas)+1]	
				h.allArenas[len(h.allArenas)-1] = ri

				// 原子性存储防止来自新堆的arena的对象变得有效在锁释放之前(这不可能发生，但是这是一个小缺点)
				atomic.StorepNoWB(unsafe.Pointer(&12[ri.l2()]), unsafe.Pointer(r))
			}

			if raceenables {
				racemapshadow(v, size)
			}

	53 runtime (l *linearAlloc) alloc(size, align uintptr, sysStat *uint64) unsaft.Pointer
			// l.next对齐align
			p := round(l.next, align)
			if p+size > l.end {
				return nil
			}

			l.next = p + size
			// pEnd 对齐physPageSize
			if pEnd := round(l.next-1, physPageSize); pEnd > l.mapped {
				// 
				sysMap(unsafe.Pointer(l.mapped), pEnd-l.mapped, sysStat)
				sysUsed(unsafe.ponter(l.mapped), pEnd-l.mapped)	
				l.mapped = pEnd
			}
			return unsafe.Pointer(p)

	54 runtime sysReserve(v unsafe.Pointer, n uintptr) unsafe.Pointer
			p, err := mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP__PRIVATE, -1, 0)
			if err != 0 {
				return nil
			}
			return p

	55 runtime sysReserveAligned(v unsafe.Pointer, size, align uintptr) (unsafe.Pointer, uintptr)
			// sysReserverAligned 类似sysReserve，但是返回的指针是对齐 align字节的。他可能预申请n或者是
			// n+align字节，所以它返回预申请的大小

			// 在本函数调用中align可能相当大，我们不可能随机获得它，所以我们要求一个更大的地区和移除我们不需要的部分
			
			retries := 0
		retry:
			p := uintptr(sysReserve(v, size+align))
			switch {
				case p == 0:
					return nil, 0
				case p&(align-1) == 0
					// 
					return unsafe.Pointer(p), size+align
				case GOOS == "windows"
					//
					sysFree(unsafe.Pointer(p), size+align, nil)
					p = round(p, align)
					p2 := sysReServe(unsafe.Pointer(p), size)
					if p != uintptr(p2) {
						sysFree(p2, size, nil)
						if retries++; retries == 100 {
							throw("")
						}
						goto retry
					}
					// 成功
					return p2, size
				default:
					// p对齐align
					pAligned := round(p, align)
					// 释放多余的部分
					sysFree(unsafe.Pointer(p), pAlign-p, nil)
					end := pAligned+size
					endLen := (p+size+align) - end
					if endLen > 0 {
						sysFree(unsafe.Pointer(end), endLen, nil)
					}
					return unsafe.Pointer(pAligned), size
			}

	56 runtime persistentalloc(size, align uintptr, sysStat *uint64) unsafe.Pointer
			// 可以申请小块的sysAlloc修饰器
			// 没有相关的释放操作
			// 针对如 函数/类型/debug相关的持续数据对象
			// 如果对齐是0，使用默认对齐
			// 返回的内存内容是0

			var p *notInHeap
			systemstack(func() {
				p = persistentalloc1(size, align, sysStat)
			})
			return unsafe.Pointer(p)
		
	57 runtime presistentalloc1(size, align uintptr, sysStat *uint64) *notInHeap
			//go:systemstack
			// 必须在系统栈运行因为stack增长会调用它
			const (
				maxBlock = 64 << 10
			)

			if size == 0 {
				throw("")
			}

			if align != 0 {
				// 不是2的幂
				if align&(align-1) != 0 {
					throw("")
				}	
				// align字段太大
				if align > _PageSize {
					throw("")
				}
			} else {
				align = 8
			}

			if size >= maxBlock {
				return (*notInHeap)(sysAlloc(size, sysStat))
			}

			mp := acquirem()
			var persistent *persistentAlloc
			if mp != nil && mp.p != 0 {
				// 获取每个p的persistentAlloc，避免锁
				persistent = &mp.p.ptr().palloc
			} else {
				lock(&globalAlloc.mutex)
				persistent = &globalAlloc.persistentAlloc
			}

			// 对齐align
			persistent.off = round(persistent.off, align)
			// persistentChunkSize = 256KB
			if persistent.off+size > persistentChunkSize || persistent.base == nil {
				persistent.bash = (*notInHeap)(sysAlloc(persistentChunkSize, &memstats.other_sys))
				if persistent.bash == nil {
					if persistent == &globalAlloc.persistentAlloc {
						unlock(&globalAlloc.mutex)
					}
					throw("")
				}

				// 添加新的chunk到persistentChunks列表
				for {
					chunks := uintptr(unsafe.Pointer(persistentChunks))
					*(*uintptr)(unsafe.Pointer(persistent.bash)) == chunks
					if atomic.Casuintptr((*uintptr)(unsafe.Pointer(&persistentChunks)), chunks, uintptr(unsafe.Pointer(persistent.base)))
						break
					}
				}	
				persistent.off = round(sys.PtrSize, align)
			}
			p := persistent.base.add(persistent.off)
			persistent.off += size
			releasem(mp)
			if persistent == &globalAlloc.persistentAlloc {
				unlock(&global.mutex)
			}

			if sysStat != &memstats.other_sys {
				mSysStatInc(sysStat, size)
				mSysStatDec(&memstat.other_sys, size)
			}

			return p

	58 runtime (h *mheap) growAddSpan(v unsafe.Pointer, size uintptr)
			// growAddSpan添加一个span当heap在[v, v+size)范围增长
			// 这个memory必须处于Prepared状态(不是Ready状态)
			//
			// h必须locked
			// 清楚一些页面来弥补我们申请的虚拟内存空间，但只是我们需要的时候

			h.scavengeIfNeededLocked(size)
			
			// 通过mheap_.spanalloc申请mspan，初始化mspan
			s := (*mspan)(h.spanalloc.alloc())
			s.init(uintptr(v), size/pageSize)
			h.setSpans(s.base(), s.npages, s)
			s.state = mSpanFree

			// [v, v+size) 总是处于Prepare状态。新的span必须标记为清除以至于申请者转换它为Ready状态当从新span申请时。
			s.scavenged = true
			// 这个span既是被释放和空闲，但是grow已经更新memstats。
			// 合并附近的mspan如果可以的话。
			h.coalesce(s)
			h.free.insert(s)

	59 runtime (h *mheap) scavengeIfNeedLocked(size uintptr)
			// scavengeIfNeededLocked 调用 scavengeLocked 如果我们现在超过scavenged目标用来阻止突变超过清理器

			// r:堆保留的空间
			if r := heapRetained(); r+uint64(size) > h.scavengeRetainedGoal {
				todo := uint64(size)
				if overage := r + uint64(size) - h.scavengeRetainedGoal; overage < todo {
					todo = overage
				}
				h.scavengeLocked(uintptr(todo))
			}

	60 runtime (h *mheap) scavengeLocked(nbytes uintptr) uintptr
			// scavengeLocked 清理空闲heap中等价于nbytes的spans 通过从拥有最高base地址的span开始，然后向下工作
			// 它随后获取这些spans和将他们放置在scav

			released := uintptr(0)
			// 首先迭代拥有大页的spans，然后
			const mask = treapIterScav | trepIterHuge
			for _, match := range []treapIterType{treapIterHuge, 0} {
				for t := h.free.end(mask, match); release < nbytes && t.valid(); {
					s := t.span()
					// 获取s的物理页边界
					start, end := s.physPageBounds()
					if start >= end {
						// 这个span没有覆盖至少一个物理页，跳过
						t = t.prev()
						continue
					}

					n := t.prev()
					if span := h.scavengeSplit(t, nbytes-released); span != nil {
						s = span
					} else {
						h.free.erase(t)
					}
					
				}
			}

			return released

	61 runtime (s *mspan) physPageBounds() (uintptr, uinptr)
			start := s.base()
			end := start + s.npages<<_PageShift
			if physPageSize > _PageSize {
				start = (start + physPageSize-1) &^ (physPageSize -1)
				end &^ = physPageSize -1
			}
			return start, end

	62 runtime (s *mheap) scavengeSplit(t treapIter, size uintptr) *mspan
			// 清理中切割
			// scavengeSplit 执行t.span()然后尝试拆分一个大小等价于物理页的span。

			// 拆分点只是通过size定义由于分割点每次是对齐于physPagesize和pageSize。如果physHugePageSize是非零然后分割点将span中的一个大页分开，接着分割点也对齐到physHugePageSize。

			// 如果期望的分割点结束于s的base，或者如果size明显大于s，接着分割是不可能的那这个奋发返回nil。否则如果分割发生，它会返回一个新的span。
			s := t.span()
			start, end := s.physPageBounds()
			if end <= start || end-start <= size {
				return nil
			}
		
			// span比我们需要的大，所以计算新的span的base如果我们决定切割。
			base := end - size
			// 对齐下个物理页或者逻辑页，看那个更大
			base &^ = (physPageSize-1) | (pageSize-1)
			if base <= start {
				return nil
			}
			if physHugePageSize >= pageSize && base&^(phyHugePage-1) >= start {
				// 我们处于分开一个大业的危险中，所以在边界上通过向下对齐大页 包含整个大页。base应该仍然对齐pageSize
				base &^= physHugePageSize -1
			}
			if base == start {
				return nil
			}
			if base < start {
				throw("")
			}

			// 从mheap_.spanalloc申请内存
			n := (*mspan)(h.spanalloc.alloc())
			nbytes := s.base() + s.npages*pageSize - base
			h.free.mutate(t, func(s *mspan) {
				n.init(base, nbytes/pageSize)
				s.npages -= nbytes/pageSize
				h.setSpan(n.base()-1, s)
				h.setSpan(n.base(), n)
				h.setSpan(n.base()+bytes-1, n)
				n.needzero = s.needzero
				n.state = s.state
			})
			return n

	63 runtime (s *mspan) released() uintptr
			// 释放返回span中返回给OS的字节数
			if !s.scavenged {
				return 0
			}
			// 获取对齐物理页的mspan的start地址和end地址
			start, end := s.physPageBounds()
			return end - start

	64 runtime stackcacherefill(c *mcache, order uint8)
			//go:systemstack
			
			// stackcacherefill/stackcacherelease实现一个全局栈帧池。这个全局池要求用来
			// 阻止per-thread caches无限制的增长。

			if stackDebug >= 1 {
				print("")
			}

			// 从全局cache中获取一些stacks
			// 抓取允许容量的一半
			var list gclinkptr
			var size uintptr
			lock(&stackpoolmu)
			for size < _StackCache/2 {
				x := stackpoolalloc(order)
				x.ptr().next = list
				list = x
				size += _FixedStack << order
			}
			unlock(&stackpoolmu)
			c.stackcache[order].list = list
			c.stackcache[order].size = size

	65 runtime release(mp *m) 
			// 获取当前协程 g
			_g_ := getg()
			mp.locks--
			if mp.locks == 0 && _g_.preempt {
				// 在我们在新栈已经清除它的情况下重新存储抢占请求
				_g_.stackguard0 = stackPreempt
			}

	66 runtime newm1(mp *m)
			if iscgo {
				var ts cgothreadstart
				if _cgo_thread_start == nil {
					throw("")
				}	
				ts.g.set(mp.g0)
				ts.tls = (*uint64)(unsafe.Pointer(&mp.tls[0]))
				ts.fn = unsafe.Pointer(funcPC(mstart))
				if msanenabled {
					msanwrite(unsafe.Pointer(&ts), unsafe.Sizeof(ts))
				}
				// 阻止进程克隆
				execLock.rlock()
				asmcgocall(_cgo_thread_start, unsafe.Pointer(&ts))
				execlock.runlock()
				return 
			}

			// 阻止进程克隆
			execLock.rlock()
			newosproc(mp)
			execLock.runlock()

	67 runtime mstart()
			//go:nosplit
			//go:nowritebarrierrec

			// mstart作为新的Ms的入口点
			// 不能分裂stack因为我们还没有设置stack边界
			// 在STM期间可能运行(因为还没有拥有P)，所以写屏障不允许

			// 获取当前协程_g_
			_g_ := getg()
		
			osStack := _g_.stack.lo == 0
			if osStack {
				// 从系统栈初始化栈边界
				// Cgo可能在stack.hi中留下栈大小
				// minit可能更新stack边界

				size := _g_.stack.hi
				if size == 0 {
					size = 8192  * sys.StackGuardMultiplier
				}
				_g_.stack.hi = uintptr()
				_g_.stack.lo = _g_.stack.hi - size + 1024
			}

			// 初始化stack guard所以我们可以开始调用常规的go代码
			_g_.stackguard0 = _g_.stack.lo + _StackGuard
			// 这个是g0，所以我们也可以调用 go:systemstack函数，它回检查stackguard1
			_g_.stackguard1 = _g_.stackguard0
			mstart1()

			// 退出这个线程
			if GOOS == "windows" || GOOS == "solaris" || GOOS == "illumos" || GOOS == "plan9" || GOOS == "darwin" || GOOS == "aix" {
				// Windows，Solaris，illumos，Darwin，AIX and Plan 9 总是系统-申请 的stack，但是在mstart之前将它放在_g_.stack中，所以以上的逻辑目前为止还没有设置osStack 
				osStack = true
			}
			mexit(true)

	68 runtime mstart1() 
			// 获取当前协程_g_
			_g_ := getg()
			if _g_ != _g_.m.g0 {
				throw()
			}	

			// 记录调用者用于在mcall中用为top stack和终止线程。
			// 在我们调用schedule之后我们从不返回到mstart1，所以其他的调用可以复用当前帧
			save(getcallerpc(), getcallersp())
			asminit()
			minit()
			
			// 安装信号控制器；在minit之后minit可以让线程准备可以控制信息
			if _g_.m == &m0 {
				mstartm0()
			}

			if fn := _g_.m.mstartfn; fn != nil {
				// 运行fn函数
				fn()
			}

			if _g_.m != &m0 {
				acquirep(_g_.m.nextp.ptr())
				_g_.m.nextp = 0
			}
			schedule()

	69 runtime save(pc, sp uintptr)
			//go:nospilt
			//go:nowritebarrierrec
			_g_ := getg()
		
			_g_.sched.pc = pc
			_g_.sched.sp = sp
			_g_.sched.lr = 0
			_g_.sched.ret = 0
			_g_.sched.g = guintptr(unsafe.Pointer(_g_))
			
			if _g_.sched.ctxt != nil {
				badctxt()
			}

	70 runtime type gobuf struct
			// sp，sp 和g的偏移量 ard known to (hard-coded in)libmach
			//
			// ctxt
			sp uintptr
			pc uintptr
			g guintptr
			ctxt unsafe.Pointer
			ret sys.Uintreg
			lr uintptr
			bp uintptr

	71 runtime minit()
			// 初始化一个新的m(包括bootstrap m)
			// 在新的协程上调用，不能申请内存	
			minitSignals()
			
			getg().m.procid = uint64(gettid())

	72 runtime minitSignals()
			// 当初始化一个新m时调用minitSignals来设置线程可供选择的signal stack和signal mask
			minitSignalStack()
			minitSignalMask()

	73 runtime minitSignalStack()
			// 当初始化一个新的m时调用minitSignalStack来设置可供选择的signal stack。如果可供选择 signal stack不是为这个协程(通常场景)，则设置这个可选择的signal stack为gsignal stack。如果这个可选择的signal stack为这个线程设置(这个场景时当一个非go 线程设置这个可选择的signal stack然后调用一个go函数) 然后设置这个gsignal为这可选择signal stack。记录在newSigStack中作出的选择，这样就能一蹴而就。
			//

			_g_ := getg()
			var st stackt
			signalstack(nil, &t)
			if st.ss_flags&&_SS_DISABLE != 0 {
				signalstack(&_g_.m.gsignal.stack)
				_g_.m.newSigstack = true
			} else {
				setGsignalStack(&st, &_g_.m.goSigStack)
				_g_.m.newSigstack = false
			}
		
	74 runtime mstartm0
			//go:yeswritebarrierrec
			// mstartm0实现了只在m0运行的mstart1的部分
			// 在线程上为callbacks创建一个额外的M 而不是通过Go来创建。在Windows上为callbacks 一个额外M另外也需要通过syscall.NewCallback创建。
			if (iscgo || GOOS == "windows") && !cgoHasExtraM {
				cgoHasExtraM = true
				newextram()
			}
			initsig(false)
		

	75 runtime initsig(preinit bool)
		//go:nosplit
		//go:nowritebarriesrrec

		// 初始化 signals
		// 通过libpreinit调用所以runtime可能还没有初始化
o
		if !preinit {
			// 现在运行signal handlers没有问题
			signalsOK = true
		}

		//
		if (isarchive || islibrary) && !preinit {
			return
		}

		for i := uint32(0); i < _NSIG; i++ {
			t := &sigtable[i]
			if t.flags == 0 || t.flags&_SigDefault != 0 {
				continue
			}

			// 我们不需要使用原子操作因为到目前为止还没有任何goroutine
			fwdSig[i] == getsig[i]

			// 是否需要安装信号控制器
			if !sigInstallGoHandler(i) {
				// 尽管我们没有安装一个信号控制器，如果需要设置SA_ONSTACK
				if fwdSig[i] != _SIG_DFL && fwdSig[i] != _SIG_IGN  {
					setsigstack(i)
				} else if fwdSig[i] == _SIG_IGN {
					sigInitIgnored(i)
				}
				continue
			}
			
			handlingSig[i] = 1
			setsig(i, funcPC(sighandler))
		}

	76 runtime getsig(i uint32) uintptr
		//go:nosplit
		//go:nowritebarrierrec

		var sa sigactiont
		sigaction(i, nil, &sa)
		return sa.sa_handler

	77 runtime sigaction(sig uint32, new, old *sigactiont)
		//go:nosplit
		//go:nowritebarrierrec

		if msanenabled && new != nil {
			msanwrite(unsafe.Pointer(new), unsafe.Sizeof(*new))
		}

		if _cgo_sigaction == nil || inForkedChild {
			sysSigaction(sig, new, old)
		} else {
			// 我们需要调用_cgo_sigaction，这就意味着我们需要一个足够大的C栈。更糟糕的是，我们可能处于libpreinit(在runtime初始化之前)或者在一个异步信号处理控制器中。(当前线程在goroutine之前切换)，g0系统栈已经在使用中。

			var ret int32
			var g *g
			if mainStarted {
				// 主线程已经开始
				g = getg()
			}

			sp := uintptr(unsafe.Pointer(&sig))
			switch {
				case g == nil :
					// No g: 我们在C栈或者信号栈
					ret = callCgoSigaction(uintptr(sig), new, old)
				case sp < g.stack.lo || sp >= g.stack.hi:
					// 我们不再处于g'stack中，所以我们必须正在处理信号。可能我们中断在g和g0之间切换的线程，所以我们应该停留在当前stack以避免
					ret = callCgoSigaction(uintptr(sig, new, old)
				default:
					// 我们正在g's stack上运行，所以要么我们不在信号控制器中，要么这个信号控制器已经设置g。如果我们运行在gsignal或者g0，systemstack将会直接调用。否则，它将切换到g0来确保我们有足够的空间来调用libc函数

					// 这个我们传给systemstack的函数迭代器不会分裂，但是这没有问题。我们将会在刷新，干净的系统栈运行所以栈检查总会成功。
					systemstack(func() {
						ret = callCgoSigaction(uintptr(sig), new, old)
					})
			}

			const EINVAL = 22
			if ret == EINVAL {
				sysSigaction(sig, new, old)
			}
		}

		if msanenabled && old != nil {
			msanread(unsafe.Pointer(old), unsafe.Sizeof(*old))
		}

	78 runtime sigInstallGoHandler(sig uint32) bool
		//go:nosplit
		//go:nowritebarrierrec

		// 对于一些信号，我们更注重一个继承SIG_IGN控制器而不是坚持我们默认的控制器。尽管这些信号可以通过使用os/signal包来获得。

		switch sig {
		case _SIGHUP, _SIGINT:
			if atomic.Loaduintptr(&fwdSig[sig]) == _SIG_IGN {
				return false
			}
		}

		t := &sigtable[sig]
		if t.flags&_SigSetStack != 0 {
			return false
		}

		// 当使用c-archive或者c-shared构建时，对于异步信号和SIGPIPE只有安装信号控制器
		if (isarchive || islibrary) && t.flags&_SigPanic == 0 && sig != _sigPIPE {
			return false
		}
	
		return true

	79 runtime setsigstack(i)
		var sa sigactiont
		sigaction(i, nil, &sa)
		if sa.sa_flags&_SA_ONSTACK != 0 {
			return 
		}
		sa.sa_flags |= _SA_ONSTACK
		sigaction(i, &sa, nil)

	80 runtime schedule()
		// scheduler运行一次：寻找一个可以执行goroutine并且执行它。从不返回
		_g_ := getg()

		if _g_.m.locks != 0 {
			throw("schedule: holding locks")
		}

		if _g_.m.lockedg != 0 {
			stoplockedm()
			execute(_g_.m.lockedg.ptr(), false)
		}

	81 runtime stoplockedm()
		// 停止由g锁定的当前m的执行，直到g可以再次可运行。
		_g_ := getg()
		
		if _g_.m.lockedg == 0 || _g_.m.lockedg.ptr() != _g_.m {
			throw("")
		}

		if _g_.m.p != 0 {
			// 调度另外一个M来运行这个P
			_p_ := releasep()
			headoffp(_p_)
		}

		incidlelocked(1)	
		//
		notesleep(&_g_.m.park)
		noteclear(&_g_.m.park)

	82 runtime handoffp(_p_ *p)
		// 通过syscall或者locked M挂起P
		// handoffp必须在任何findrunnable返回一个G用来在_p_上运行的场景下启动一个M
		
		// 如果它有本地工作，直接启动它
		if !runqempty(_p_) || shed.runqsize != 0 {
			startm(_p_, false)
			return
		}

		// 如果它由GC工作，直接启动它
		if gcBlackenEnabled != 0 && gcMarkWrokavailable(_p_) {
			startm(_p_, false)
			return 
		}

		// 没有本地工作，检查是否由空闲或者自旋的M,否则不需要我们的帮助
		if atomic.Load(&sched.nmspinning) + atomic.Load(&sched.npidle) == 0 && atomic.Cas(&sched.nmspinning, 0, 1) {
			startm(_p_, true)
			return
		}

		lock(&sched.lock)
		// gc等待运行
		if sched.gcwaiting != 0 {
			_p_.status = _pgcstop
			sched.stopwait--
			if sched.stopwait == 0 {
				notewakeup(&sched.stopnote)
			}
			unlock(&sched.lock)
			return
		}

		if _p_.runSafePointFn != 0 && atomic.Cas(&_p_.runSafePointFn, 1, 0) {
			sched.safePointFn(_p_)
			sched.safePointWait--
			if sched.safePointWait == 0 {
				notewakeup(&sched.stopnote)
			}
		}

		if sched.runqsize != 0 {
			unlock(&sched.lock)
			startm(_p_, false)
			return
		}

		if sched.npidle == uint32(gomaxprocs-1) && atomic.Load64(&sched.lastpoll) != 0 {
			unlock(&sched.lock)
			startm(_p_, false)
			return
		}

		if when := nobarrierWakeTime(_p_); when != 0 {
			wakeNetPoller(when)
		}
		pidleput(_p_)
		unlock(&sched.lock)


	83 runtime runqempty(_p_ *p) bool
		// runqempty 获取_p_在它的本地运行队列是否有G。
		// 它永远不会虚假的返回true

		for {
			head := atomic.Load(&_p_.runqhead)
			tail := atomic.Load(&_p_.runqtail)
			runnext := atomic.Loaduintptr((*uintptr)(unsafe.Pointer(&_p_.runnext)))
			if tail == atomic.Load(&_p_.runtail) {
				return head == tail && runnext == 0
			}
		}

		
	84 runtime newosproc(mp *m)
		stk := unsafe.Pointer(mp.g0.stack.hi)
		// 如果我们在这里使用CLONE_PTRACE,strace会感到困惑

		if false {
			print("")
		}
		
		// 在clone期间禁止信号，所以新的线程启动伴随着信号无效。在minit中使信号生效。
		var oset sigset
		sigprocmask(_SIG_SETMASK, &sigset_all, &oset)
		// cloneFlags = _CLONE_VM | _CLONE_FS | _CLONE_FILES | _CLONE_SIGHAND | _CLONE_SYSVSEM | _CLONE_THREAD
		ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
		sigprocmask(_SIG_SETMASK, &oset, nil)

		if ret < 0 {
			print("")
			if ret == -_EAGAIN {
				println()
			}
			throw("newosproc")
		}

	85 runtime pidleput(_p_ *p)
		// 将p放在_Pidle列表中
		// Sched必须被锁
		// 可能在STW中运行，所以写屏障不允许

		if !runqempty(_p_ *p) {
			throw("")
		}

		// 将_p_.link设置为sched的空闲p
		_p_.link = sched.pidle
		// sched设置空闲p
		sched.pidle.set(_p_)
		// sched空闲计算加1
		atomic.Xadd(&sched.npidle, 1)

	86 runtime incidlelocked(v int32)
		lock(&sched.lock)
		sched.nmidlelocked+= v
		if v > 0 {
			checkdead()
		}
		unlock(&sched.lock)

	87 runtime checkdead()
		// 检查deadlock场景
		// 这个检查是基于正在运行的M的数量，如果 0 -> deadlock		
		// sched.lock必须持有
		
		// 对于-buildmode=c-shared 或者 -buildmode=c-archive 这没有问题如果没有正在运行的goroutines。
		// 如果由于在已经空闲的线程中捕获一个信号 我们快要消失
		// freezetheworld 将导致所有运行的线程处于阻塞
		// runtime将本质上进入deadlock的状态会调用 除非有一个线程很快会调用exit
		
		if panicking > 0 {
			return
		}
		
		var run0 int32
		if !iscgo && cgoHasExtraM {
			mp := lockextra(true)
			haveExtraM := extraMCount > 0
			unlockextra(mp)
			if haveExtraM {
				run0 = 1
			}
		}
		
		run := mcount() - sched.nmidle - sched.nmidlelocked - sched.nmsys
		ifrun > run0 {
			return
		}

		if run < 0 {
			print("")
			throw()
		}

		grunning := 0
		lock(&allglock)
		for i := 0; i < len(allgs); i++ {
			gp := allgs[i]
			if isSystemGoroutine(gp, false) {
				continue
			}

			s := readgstatus(gp)
			switch s &^ _Gscan {
				case _Gwaiting,
					_Gpreempted:
					grunning++
				case _Grunnable,
					_Grunning,
					_Gsyscall:
					unlock(&allglock)
					print("")
					throw("")
			}
		}
		unlock(&allglock)
		if grunning == 0 {
			// 如果main goroutine调用runtime.Goexit()则可能
			// 解锁所以GODEBUG=scheddetail=1 不会挂起
			unlock(&sched.lock)
			throw("")
		}

		// 可能为了playgroud将时间提前
		if faketime != 0 {
			when, _p_ := timeSleepUntil()	
		}

		

	88 runtime isSystemGoroutine(gp *g, fixed bool)	
		// isSystemGoroutine确认协程g在堆栈转存和死锁检查中是省略。任何协同协程开始于runtime.*的入口点，除了runtime.main, runtime.handleAsyncEvent(只是汇编)和有时runtime.runfinq

		// 如果fixed是true,在用户和系统之间变化的任何goroutine被认为是用户协程
		f := findfunc(gp.startpc)
		if !f.valid() {
			return false
		}

		if f.funcID == funcID_runtime_main || f.funcID == funcID_handleAsyncEvent {
			return false
		}
		
		if f.funcID == funcID_runfinq {
			if fixed {
				return false
			}	
			return !fingRunning
		}
		return hasPrefix(funcname(f), "runtime.")

	89 runtime findfunc(pc uintptr) 
		datap := findmoduledatap(pc)
		if datap == nil {
			return funcInfo{}
		}

		const nsub = uintptr(len(findfuncbucket{}.subbuckets))

		x := pc - datap.minpc
		b := x / pcbucketsize
		i := x % pcbucketsize / (pcbucketsize / nsub)

		ffb := (*findfuncbucketa)(add(unsafe.Pointer(datafunctab), b*unsafe.Sizeof(findfuncbucket{})))
		idx := ffb.idx + uint32(ffb.subbuckets[i])

		//
		//
		if idx >= uint32(len(datap.ftab)) {
			idx = uint32(len(datap.ftab) -1)
		}
		
		if pc < datap.ftab[idx].entry {
			for datap.fab[idx].entry > pc && idx > 0 {
				idx--
			}	
			if idx == 0 {
				throw("")
			}
		} else {
			for datap.ftab[idex+1].entry <= pc {
				idx++
			}
		}

		funcoff := datap.ftab[idx].funcoff
		if funcoff == ^uintptr(0) {
			//
			return funcInfo{}
		}
		return funcInfo{(*_func)(unsafe.Pointer(&datap.pclntable[funcoff])), datap}

	90 runtime findmoduledatap(pc uintptr)
		// 遍历所有的moduledata;找出对应的datap
		for datap := &firstmoduledata; data != nil; data = datap.next {
			if datap.minpc <= pc && pc < datap.maxpc {
				return datap
			}
		}
		return nil

	91 runtime type moduledata struct {
		// moduledata记录可执行镜像布局的信息，它由链接器写入。这里的任何改动必须匹配在cmd/internal/ld/symtabl.go:sysmtab代码的改动. moduledata存储再静态分配的非指针的内存中;这里的指针对于垃圾收集器是不可见的。
			pclntable []byte
			ftab []functab
			filetab []uint32
			findfunctab uintptr
			minpc, maxpc uintptr

			text, etext uintptr
			noptrdata, enoptrdata uintptr
			data, edata uinptr
			bss, ebss uinptr
			noptrbss, enoptrbss uintptr
			end, gcdata, gcbss uintptr
			types, etypes uintptr

			textsectmap []textsect
			typelinks []int32
			itablinks []*itab

			ptab []ptabEntry
			pluginpath string
			pkghashes []modulehash

			modulename string
			modulehashes []modulehash

			hasmain uint8

			gcdatamask, gcbssmask bitvector

			typemap map[typeOff]*_type

			bad bool

			next *moduledata
		}			
		  
	92 runtime timeSleepUntil() 
		next := int64(maxWhen)	
		var pret *p

		// 阻止allp切片返回变化。这像retake
		lock(&allpLock)
		for _, pp := range allp {
			if pp == nil {
				// 这可能发生如果procresize扩充的allp但是还没有创建新的Ps
				continue
			}

			c := atomic.Load(&pp.adjustTimers)
			if c == 0 {
				w := int64(atomic.Load64(&pp.timer0When))
				if w != 0 && w < next {
					next = w
					pret = pp
				}
				continue
			}

			lock(&pp.timesLock)

			for _, t := range pp.timers {
				switch s := atomic.Load(&t.status); s {
					case timerWaiting:
						if t.when < next {
							next = t.when
						}
					case timerModifiedEarlier, timerModifiledLater:
						if t.nextwhen < next {
							next = t.nextwhen
						}
						if s == timerModifiedEarliser {
							c--
						}
				}

				//
				if int32(c) <= 0 {
					break
				}
			}
			unlock(&pp.timersLock)
		}
		unlock(&allpLock)
		
		return next, pret

	43 runtime.newproc1(fn *funcval, argp *uint8, narg int32, callergp *g, callerpc uintptr)
		// 创建一个运行fn从argp开始拥有narg字节参数的协程。callerpc是创建它的go语句地址。
		// 新的g被放到等待运行的g队列中。
		_g_ := getg() //获取当前goroutine
		acquirem() // 

		_p_ := _g_.m.p.ptr() //返回PP指针
		newg := gfget(_p_)

	44 runtime.gfget(_p_ *p) *g
		// 从g空闲列表中获取
		// 如果本地列表是空，则从全局列表中获取一批

		retry:
		if _p_.gFree.empty() && (!sched.gFree.stack.empty() || !sched.gFree.noStack.empty()) {
			lock(&sched.gFree.lock)
			//将一批空闲的Gs移到P
			if _p_.gFree.n < 32 {
				//优先获取有栈的Gs，同时如果 sched.gFree.stack.head = gp.schedlink
				gp := sched.gFree.stack.pop()
				if gp == nil {
					//
					gp = sched.gFree.noStack.pop()	
					if gp == nil {
						break
					}
				}
				sched.gFree.n--
				_p_.gFree.push(gp) //以入栈的方式将goroutine 连接起来
				_p_.gFree.n++ 
			}
			unlock(&sched.gFree.lock)
			goto retry
		}

		gp := _p_.gFree.pop() //弹出最后进栈的goroutine
		if gp == nil {
			return nil
		}

		_p_.gFree.n--
		if gp.stack.lo == 0 {
			// stack 已经在 gfput 中回收，重新申请一个新的
			systemstack(func() {
				gp.stack = stackalloc(_fixedStack)	
			})
			gp.stackguard0 = gp.stack.lo + _StackGurad	
		} else {
			if raceenabled {
				racemalloc(unsafe.Pointer(gp.stack.lo), gp.stack.hi-gp.stack.lo)
			}
			if msanenabled {
				msanmalloc(unsafe.Pointer(gp.stack.lo), gp.stack.hi-gp.stack.lo)
			}	
		}

		return gp

	45 runtime.stackalloc(n uint32) stack
		// 申请b个自己的栈
		// stackalloc必须运行在系统栈上面因为它用每个P的资源，并且不能缩放堆栈。
		// Stackalloc必须运行在scheduler栈，因为我们从不尝试在stackalloc运行的代码期间
		// 增加堆栈 


### 参考
	https://docs.oracle.com/cd/E19205-01/820-1200/blaoy/index.html
	https://blog.csdn.net/u010853261/article/details/103359762
	https://me.csdn.net/u010853261	
	http://man7.org/linux/man-pages/man7/futex.7.html
	https://github.com/golang/proposal/blob/master/design/35112-scaling-the-page-allocator.md
