入门
========

为了使用Ray,你需要理解以下两点:

- Ray是怎样异步地执行任务达到并行性的.
- Ray是怎样使用对象ID代表不可变的远端对象的.

概览
--------

Ray是一个基于Python的分布式执行引擎. 同样的代码能够在单机上通过多进程的方式提升性能,
同时也可以在集群环境运行以获取更大算力.

当使用Ray时,多个进程会参与其中.

- 多个 **worker** 进程处理任务执行并将结果存入到 **object store** 中.
  每一个worker都是一个单独的进程.
- 每个节点均有一个 **object store** 进程将不可变对象存入到共享内存中,
  并允许在同一个节点上的多个 **worker** 进程之间以最小的拷贝和反序列化代价共享对象.
- 每个节点均有一个 **local scheduler** 将任务分配到该节点的不同 **worker** 上执行.
- 全局唯一的 **global scheduler** 负责将 **local scheduler** 发送来的任务
  分配到别的 **local scheduler** 上.
- 用户控制的Python进程被称为 **driver** .例如,一个用户在Python shell上运行了一段脚本，
  那么运行这段脚本的Python进程或者这个shell程序就是 **driver**. 一个 **driver** 进程
  与 **worker** 进程一样可以提交任务到 **local scheduler** 并且可以读取
  **object store** 中的对象, 但与 **worker** 不同的是, **local scheduler** 不会分配任务
  到 **driver** 执行.
- 一个 **Redis server** 进程维护了大部分的系统状态.例如, 它对哪个对象和任务(但不包含数据)
  位于哪个机器进行了追踪. 在调试时, 它可以被直接查询.

启动Ray
------------

为了启动Ray, 首先启动Python并运行如下命令.

.. code-block:: python

  import ray
  ray.init()

这样Ray就被启动了.

不可变远端对象
------------------------

在Ray中, 我们可以创建对象并在对象上进行计算. 我们称这些对象为 **remote objects**,
我们使用 **object IDs** 来引用它们. 这些 **remote objects** 被存入 **object stores**,
并且集群中的每个节点都有一个 **object store**. 在集群环境中时, 我们或许并不知道
每一个对象到底位于哪一台机器上.

每一个远端对象都需要有一个唯一的 **object ID** 来引用它. 如果你熟悉Futures, 会发现
我们的对象ID是一个相似的概念.

我们假定所有的远端对象都是不可变的. 也就是说, 它们的值在创建之后不能被改变.
这允许远端对象能够在多个object store中被复制并且不需要任何同步操作.

Put和Get操作
~~~~~~~~~~~

命令 ``ray.get`` 和 ``ray.put`` 能被用来在Python对象和对象ID之间进行转换, 如下例所示.

.. code-block:: python

  x = "example"
  ray.put(x)  # ObjectID(b49a32d72057bdcfc4dda35584b3d838aad89f5d)

命令 ``ray.put(x)`` 将会被worker进程或driver进程(driver进程是运行用户脚本的进程)调用.
它以一个Python对象作为输入并将其拷贝至本地的object store中(这里本地指的是相同节点).
一旦对象被存入到object store中, 它的值将不能改变.

此外, ``ray.put(x)`` 返回一个对象ID, 本质上是一个用来访问新创建的远端对象的ID.
如果我们用 ``x_id = ray.put(x)`` 将对象ID存入一个变量中, 我们可以将 ``x_id`` 传给远端函数,
从而可以在远端函数操作相应的远端对象.

命令 ``ray.get(x_id)`` 以对象ID作为参数, 从对应的远端对象创建一个Python对象.
对于一些如数组的对象, 我们可以使用共享内存避免对象拷贝. 对于其它对象, 这个命令将
对象从object store中拷贝至worker进程的堆中. 如果 ``x_id`` 对应的远端对象与调用 ``ray.get(x_id)``
的worker进程不在同一个节点上, 那么远端对象首先会被从它目前所在的object store传送至需要的object store上.

.. code-block:: python

  x_id = ray.put("example")
  ray.get(x_id)  # "example"

如果 ``x_id`` 对应的远端对象还没有被创建出来, 那么 ``ray.get(x_id)`` 会一直等待到远端对象
被创建出来.

一个常见的 ``ray.get`` 的用例是获取一个对象ID列表的远端对象. 在这个用例中, 你可以
调用 ``ray.get(object_ids)`` , 这里 ``object_ids`` 是一个对象ID的列表.

.. code-block:: python

  result_ids = [ray.put(i) for i in range(10)]
  ray.get(result_ids)  # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

Ray中的异步计算
-------------------------------

Ray可以使任意的Python函数异步执行. 这是通过将一个Python函数指明为一个 **远端函数(remote function)** 实现的.

例如, 一个普通的Python函数通常长这样.

.. code-block:: python

  def add1(a, b):
      return a + b

一个远端函数则长这样.

.. code-block:: python

  @ray.remote
  def add2(a, b):
      return a + b

远端函数
~~~~~~~~~~~~~~~~

但是两者的调用过程是不一样的, 当调用 ``add1(1, 2)`` 返回 ``3`` 时, Python解释器
会阻塞直到计算完成, 但当调用 ``add2.remote(1, 2)`` 时, 一个对象ID会被立即返回同时
一个 **任务(task)** 会被创建出来. 这个任务会被系统调度并异步地执行(可能在一个不同节点).
当任务完成后, 它的返回值会被存入object store中.

.. code-block:: python

  x_id = add2.remote(1, 2)
  ray.get(x_id)  # 3

下面的简单例子说明了异步任务是被如何用来做并行计算的.

.. code-block:: python

  import time

  def f1():
      time.sleep(1)

  @ray.remote
  def f2():
      time.sleep(1)

  # 下面的任务需要10秒完成.
  [f1() for _ in range(10)]

  # 下面的任务需要1秒完成 (假设系统有至少10个CPU).
  ray.get([f2.remote() for _ in range(10)])

这里 *提交任务 (submitting a task)* 与 *执行任务 (executing the task)* 有一个显著的区别.
当一个远端函数被调用时, 执行这个函数的任务被提交到一个local scheduler上,
任务输出的对象ID被立即返回. 然而直到系统将任务分配到某个worker上之前任务都不会被执行.
任务的执行 **不是** 懒惰的. 系统将输入数据移动到task所在节点, 然后task会在输入数据可用
并且有足够的资源后立即执行.

**当一个任务被提交后, 其所有的参数将会被以值或对象ID的方式传入.** 例如, 下面几行代码有一样的效果.

.. code-block:: python

  add2.remote(1, 2)
  add2.remote(1, ray.put(2))
  add2.remote(ray.put(1), ray.put(2))

远端函数永远不返回真实值, 只返回对象ID.

当远端函数被执行时, 它在Python对象上进行相应的操作.
也即, 如果远端函数中有一些对象ID, 系统将会从object store中获取对应的对象.

注意一个远端函数可以返回多个对象ID.

.. code-block:: python

  @ray.remote(num_return_vals=3)
  def return_multiple():
      return 1, 2, 3

  a_id, b_id, c_id = return_multiple.remote()

表达任务间的依赖关系
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

程序员可以通过传递一个任务输出的对象ID作为另一个任务的参数来表达任务之间的依赖关系.
如下, 我们创建了三个任务, 每一个任务依赖于前一个任务.

.. code-block:: python

  @ray.remote
  def f(x):
      return x + 1

  x = f.remote(0)
  y = f.remote(x)
  z = f.remote(y)
  ray.get(z) # 3

第二个任务在第一个任务完成前不会被执行, 同时第三个任务在第二个任务完成前也不会被执行.
在这个例子中, 没有机会进行并行化处理.

任务依赖输入的这种能力能够非常简单的表达复杂的依赖关系.
下面是一个树状reduce操作的实现.

.. code-block:: python

  import numpy as np

  @ray.remote
  def generate_data():
      return np.random.normal(size=1000)

  @ray.remote
  def aggregate_data(x, y):
      return x + y

  # 随机生成一些数据. 启动100个将会调度在不同节点上的任务.
  # 结果数据将分布在整个集群中.
  data = [generate_data.remote() for _ in range(100)]

  # 执行树状reduce操作.
  while len(data) > 1:
      data.append(aggregate_data.remote(data.pop(0), data.pop(0)))

  # 获取计算结果.
  ray.get(data)

远端函数中的远端函数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

到目前为止, 我们仅在driver中调用了远端函数.
但是worker进程同样可以调用远端函数.
为了说明这一点, 考虑下面的例子.

.. code-block:: python

  @ray.remote
  def sub_experiment(i, j):
      # 为第i次实验运行第j次子实验
      return i + j

  @ray.remote
  def run_experiment(i):
      sub_results = []
      # 启动任务并行处理10个子实验
      for j in range(10):
          sub_results.append(sub_experiment.remote(i, j))
      # 返回子实验返回值的和
      return sum(ray.get(sub_results))

  results = [run_experiment.remote(i) for i in range(5)]
  ray.get(results) # [45, 55, 65, 75, 85]

当远端函数 ``run_experiment`` 在一个worker上被执行时, 它调用了
另一个远端函数 ``sub_experiment`` 许多次. 这是一个多组实验的例子, 每一个实验被并行的执行,
同时实验内部也利用了系统并行性.
