Ray API
===========

启动Ray
------------

Ray可以用两种方式启动. 第一种方式你可以在一个脚本内启动所有Ray相关的进程然后关闭它们.
第二种方式你可以连接并使用一个已有的Ray集群.

脚本启动关闭集群
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

一个用例是当你调用 ``ray.init`` 的时候启动所有的Ray相关进程然后在脚本结束后关闭所有
进程. 这些进程包括本地和全局的调度器, 一个object store和一个object manager, 一个redis服务, 
以及其它进程.

**注意:** 这种方式仅限于单个机器.

如下所示操作.

.. code-block:: python

  ray.init()

如果这台机器存在GPU, 你可以使用 ``num_gpus`` 参数指定. 类似的, 你也可以用
``num_cpus`` 参数指定CPU的数目.

.. code-block:: python

  ray.init(num_cpus=20, num_gpus=2)

默认情况下, Ray将会使用 ``psutil.cpu_count()`` 决定CPU的个数. 
Ray也会试图自动决定GPU的个数.

不同于用每个节点上worker进程的数量考虑问题, 我们倾向于在每个节点的CPU和GPU资源的数量
上考虑问题, 这样可以假设我们拥有一个无限worker的池. 任务会根据资源的可用性
而不是目前可用的worker数目分配到各个worker上, 这样做可以避免资源的竞争.

连接到已有集群
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

一旦Ray集群启动后, 你为了连上它唯一需要的信息就是集群中Redis服务的地址.
在这种用例中, 你的脚本将不会启动或者关闭任何进程. 这个集群以及它所有的进程
都将被许多脚本和用户共用. 你只需要知道集群Redis服务的地址便可以连上它.
所需的命令如下所示.

.. code-block:: python

  ray.init(redis_address="12.345.67.89:6379")

在这种情况下, 你不能在 ``ray.init`` 中指定 ``num_cpus`` 或者 ``num_gpus``,
因为这些信息是在集群启动的时候被传入的, 而不是你启动脚本的时候.

阅读如何在多个节点上 `启动Ray集群`_ 的命令.

.. _`启动Ray集群`: http://ray.readthedocs.io/en/latest/using-ray-on-a-cluster.html

.. autofunction:: ray.init

定义一个远端函数
-------------------------

远端函数被用来创建任务. ``@ray.remote`` 装饰器被放到函数定义前, 用来定义一个远端函数.

远端函数可以用 ``f.remote`` 的方式调用. 调用远端函数将会创建一个 **任务** 然后被调度到
Ray集群中某个worker执行. 调用将会返回一个 **对象ID** (本质上是一个future), 代表任务
最终返回的值. 任何人拿到对象ID后可以获取它的值, 而不用去管这个任务是在哪里执行的
(参见 `从对象ID获取值`_).

当一个任务执行时, 它的输出将会被序列化成字节数组然后存入到ojbect store中.

注意远端函数的参数可以是值也可以是对象ID.

.. code-block:: python

  @ray.remote
  def f(x):
      return x + 1

  x_id = f.remote(0)
  ray.get(x_id)  # 1

  y_id = f.remote(x_id)
  ray.get(y_id)  # 2

如果你想让一个远端函数返回多个对象ID, 你可以传入 ``num_return_vals`` 参数到远端函数装饰器中.

.. code-block:: python

  @ray.remote(num_return_vals=2)
  def f():
      return 1, 2

  x_id, y_id = f.remote()
  ray.get(x_id)  # 1
  ray.get(y_id)  # 2

.. autofunction:: ray.remote

从对象ID获取值
------------------------------

对象ID可以通过调用 ``ray.get`` 转换成对象. 注意 ``ray.get`` 可以接受
单个对象ID或者一组对象ID列表作为参数.

.. code-block:: python

  @ray.remote
  def f():
      return {'key1': ['value']}

  # 获取单个对象ID.
  ray.get(f.remote())  # {'key1': ['value']}

  # 获取一个对象ID列表.
  ray.get([f.remote() for _ in range(2)])  # [{'key1': ['value']}, {'key1': ['value']}]

Numpy数组
~~~~~~~~~~~~

Numpy数组相比其他数据类型会被更高效地处理, 因此 **尽可能使用numpy数组**.

任何作为序列化对象一部分的numpy数组将不会被从object store中拷贝出去.
它们将留在object store中, 而反序列的对象将会有一个指针指向object store内存的对应位置.

因为在object store中的对象是不可变的, 这也意味着如果你想改变一个远端函数返回的numpy数组,
你必须要首先拷贝它.

.. autofunction:: ray.get

往object store中放入对象
-----------------------------------

对象被放入到object store中的主要方式是任务返回值放入. 但同样可以直接使用 ``ray.put`` 将
对象放入到object store中.

.. code-block:: python

  x_id = ray.put(1)
  ray.get(x_id)  # 1

使用 ``ray.put`` 的主要原因是你想在一些任务之间传递同一个大对象.
通过首先调用 ``ray.put`` 然后传递对象ID到各个任务, 大对象仅被拷贝了一次进入object store,
但当你直接传递对象的时候, 会被拷贝很多次.

.. code-block:: python

  import numpy as np

  @ray.remote
  def f(x):
      pass

  x = np.zeros(10 ** 6)

  # 方法1: 这里x被拷贝了10次到object store
  [f.remote(x) for _ in range(10)]

  # 方法2: 这里x仅被拷贝了1次到object store
  x_id = ray.put(x)
  [f.remote(x_id) for _ in range(10)]

Note that ``ray.put`` is called under the hood in a couple situations.
注意 ``ray.put`` 在一些情况下才会被调用.

- 一个任务返回值的时候会被调用.
- 在作为参数传入到一个任务中的时候会被调用, 除非参数类型是Python原生类型例如整数、短字符串、列表、元祖或者字典.

.. autofunction:: ray.put

等待部分任务完成
---------------------------------------

我们经常会期望当不同的任务完成时进行相应的计算. 例如, 有一堆任务拥有变长的执行时间,
它们的结果可以被任意顺序进行处理, 那么以它们完成的时间作为处理顺序是合理的.
在其它设定下, 丢弃掉结果不再需要的落后的任务或许也是合理的.

为了实现上述目的, 我们引入了 ``ray.wait`` 原语, 将一组对象ID作为参数并当一部分对象值可用后返回.
默认的, 它将阻塞并直到一个对象值可用, 但是可以用 ``num_returns`` 参数指定等待的对象值数目.
如果一个 ``timeout`` 参数被传入, 那么它将被阻塞至多那么多毫秒然后返回一个列表小于 ``num_returns``
个元素.

``ray.wait`` 函数返回两个列表. 第一个列表是一个可用的对象ID列表(最大长度是 ``num_returns``),
第二个列表是剩下的对象ID列表, 因此两个列表的合集等于传入 ``ray.wait`` 的列表(取决于顺序).

.. code-block:: python

  import time
  import numpy as np

  @ray.remote
  def f(n):
      time.sleep(n)
      return n

  # 启动3个不同时长的任务.
  results = [f.remote(i) for i in range(3)]
  # 阻塞直到其中的2个完成.
  ready_ids, remaining_ids = ray.wait(results, num_returns=2)

  # 启动5个不同时长的任务.
  results = [f.remote(i) for i in range(3)]
  # 阻塞直到其中的4个完成或者2.5秒.
  ready_ids, remaining_ids = ray.wait(results, num_returns=4, timeout=2500)

很容易使用这种构成创建一个许多任务被执行的有限循环, 并且当一个任务完成后, 启动一个新任务.

.. code-block:: python

  @ray.remote
  def f():
      return 1

  # 启动5个任务.
  remaining_ids = [f.remote() for i in range(5)]
  # 当一个任务完成后, 启动一个新任务.
  for _ in range(100):
      ready_ids, remaining_ids = ray.wait(remaining_ids)
      # 当有一个对象可用后对其进行处理.
      print(ray.get(ready_ids))
      # 启动一个新任务.
      remaining_ids.append(f.remote())

.. autofunction:: ray.wait

查看错误
--------------

在集群环境不同的进程间追踪错误是一个有挑战的任务. 有一些机制可以帮助我们做这件事.

1. 如果一个任务抛出异常, 那么异常信息将会被打印到driver进程的后台.

2. 如果 ``ray.get`` 依赖的对象ID所在的父任务在创建对象之前抛出异常, 那么异常将会被 ``ray.get`` 往上抛出.

错误信息将会在Redis中聚集并且能够使用 ``ray.error_info`` 访问. 通常你不需要这么做, 但是的确是可以的.

.. code-block:: python

  @ray.remote
  def f():
      raise Exception("This task failed!!")

  f.remote()  # 一个错误消息将会在后台打印.

  # 等待错误传播到Redis.
  import time
  time.sleep(1)

  ray.error_info()  # 返回一个错误信息列表.

.. autofunction:: ray.error_info
