  尽管Python 完全支持多线程编程，但是解释器的C 语言实现部分在完全并行执行
时并不是线程安全的。实际上，解释器被一个全局解释器锁保护着，它确保任何时候
都只有一个Python 线程执行。GIL 最大的问题就是Python 的多线程程序并不能利用
多核CPU 的优势（比如一个使用了多个线程的计算密集型程序只会在一个单CPU 上
面运行）。
  在讨论普通的GIL 之前，有一点要强调的是GIL 只会影响到那些严重依赖CPU
的程序（比如计算型的）。如果你的程序大部分只会设计到I/O，比如网络交互，那么
使用多线程就很合适，因为它们大部分时间都在等待。实际上，你完全可以放心的创
建几千个Python 线程，现代操作系统运行这么多线程没有任何压力，没啥可担心的。
而对于依赖CPU 的程序，你需要弄清楚执行的计算的特点。例如，优化底层算法
要比使用多线程运行快得多。类似的，由于Python 是解释执行的，如果你将那些性能
瓶颈代码移到一个C 语言扩展模块中，速度也会提升的很快。如果你要操作数组，那
么使用NumPy 这样的扩展会非常的高效。最后，你还可以考虑下其他可选实现方案，
比如PyPy，它通过一个JIT 编译器来优化执行效率（不过在写这本书的时候它还不
能支持Python 3）。
  还有一点要注意的是，线程不是专门用来优化性能的。一个CPU 依赖型程序可能
会使用线程来管理一个图形用户界面、一个网络连接或其他服务。这时候，GIL 会产
生一些问题，因为如果一个线程长期持有GIL 的话会导致其他非CPU 型线程一直等
待。事实上，一个写的不好的C 语言扩展会导致这个问题更加严重，尽管代码的计算
部分会比之前运行的更快些。
  你可以使用multiprocessing 模块来创建一个进程池，并像协同处理器一样的使用它.
  另外一个解决GIL 的策略是使用C 扩展编程技术。主要思想是将计算密集型任务
转移给C，跟Python 独立，在工作的时候在C 代码中释放GIL。这可以通过在C 代
码中插入下面这样的特殊宏来完成