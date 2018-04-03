Actors
======

Ray中的远端函数可以认为是纯函数式没有副作用的.
限制我们仅仅使用远端函数可以让我们获得分布式函数编程的能力,
这对于很多用例非常好, 但在实际使用中会有一些受限.

Ray用 **actor** 扩展了数据流模型. 一个actor本质上是一个有状态的worker(或者一个服务).
当一个新的actor被初始化时, 一个新的worker会被创建并且actor的方法将会被调度到这个特定的worker上,
并且actor的方法可以获取和改变worker的状态.

假定我们已经启动Ray.

.. code-block:: python

  import ray
  ray.init()

定义和创建一个actor
------------------------------

考虑下面一个简单的例子. 装饰器 ``ray.remote`` 表示 ``Counter`` 对象的实例将会是actor.

.. code-block:: python

  @ray.remote
  class Counter(object):
      def __init__(self):
          self.value = 0

      def increment(self):
          self.value += 1
          return self.value

为了真正创建一个actor, 我们可以通过调用 ``Counter.remote()`` 实例化这个类.

.. code-block:: python

  a1 = Counter.remote()
  a2 = Counter.remote()

当一个actor被实例化时, 会发生下面的事件:

1. 集群中的一个节点被选中, 一个worker进程被创建到这个节点(通过节点自身的local scheduler)用于运行actor调用的方法.
2. 一个 ``Counter`` 对象被创建到上述worker同时 ``Counter`` 的构造函数被调用.

使用actor
--------------

我们可以通过调用actor自身方法的方式调度任务到这个actor.

.. code-block:: python

  a1.increment.remote()  # ray.get returns 1
  a2.increment.remote()  # ray.get returns 1

当 ``a1.increment.remote()`` 被调用时, 会发生下面的事件:

1. 一个任务被创建.
2. 这个任务被driver的local scheduler直接分配到actor所在节点的local scheduler.
   也就是说调度过程绕过了global scheduler.
3. 一个对象ID被返回.

我们接着可以通过对象ID调用 ``ray.get`` 来获取对象的值.

类似的, 调用 ``a2.increment.remote()`` 会产生一个任务并被直接调度到第二个 ``Counter`` 的actor上.
因为这两个任务跑在不同的actor上,
它们可以被并行的执行(注意只有actor的方法会被调度到一个actor类型的worker上, 而通常的远端函数不会).

另一方面, 相同 ``Counter`` actor上的方法调用会以它们被调用的顺序串行地执行.
因此它们可以彼此之间分享状态, 如下所示.

.. code-block:: python

  # 创建10个Counter类型的actor.
  counters = [Counter.remote() for _ in range(10)]

  # 增加每个Counter一次并获取结果. 这些任务都是并行运行的.
  results = ray.get([c.increment.remote() for c in counters])
  print(results)  # prints [1, 1, 1, 1, 1, 1, 1, 1, 1, 1]

  # 增加第一个Counter 5次. 这些任务被串行执行并且彼此分享状态.
  results = ray.get([counters[0].increment.remote() for _ in range(5)])
  print(results)  # prints [2, 3, 4, 5, 6]

一个更有趣的actor例子
--------------------------------

一个常见的actor使用模式是用其封装外部库或外部服务管理的可变状态.

`Gym`_ 为许多测试和训练强化学习agents的模拟环境提供了一组接口.
这些模拟器是有状态的, 并且使用这些模拟器的任务必须修改它们的状态.
我们可以使用actor封装这些模拟器的状态.

.. _`Gym`: https://gym.openai.com/

.. code-block:: python

  import gym

  @ray.remote
  class GymEnvironment(object):
      def __init__(self, name):
          self.env = gym.make(name)
          self.env.reset()

      def step(self, action):
          return self.env.step(action)

      def reset(self):
          self.env.reset()

我们可以初始化一个actor然后以如下方式调度任务到这个actor.

.. code-block:: python

  pong = GymEnvironment.remote("Pong-v0")
  pong.step.remote(0)  # 在模拟器中采用action 0.

Actor上使用GPU
--------------------

一个常见的actor用例是用其包含一个神经网络.
例如, 假定我们已经import了Tensorflow并且创建了一个方法用于构建神经网络.

.. code-block:: python

  import tensorflow as tf

  def construct_network():
      x = tf.placeholder(tf.float32, [None, 784])
      y_ = tf.placeholder(tf.float32, [None, 10])

      W = tf.Variable(tf.zeros([784, 10]))
      b = tf.Variable(tf.zeros([10]))
      y = tf.nn.softmax(tf.matmul(x, W) + b)

      cross_entropy = tf.reduce_mean(-tf.reduce_sum(y_ * tf.log(y), reduction_indices=[1]))
      train_step = tf.train.GradientDescentOptimizer(0.5).minimize(cross_entropy)
      correct_prediction = tf.equal(tf.argmax(y,1), tf.argmax(y_,1))
      accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))

      return x, y_, train_step, accuracy

如下所示, 我们接着为这个网络定义一个actor.

.. code-block:: python

  import os

  # 定义一个运行在GPU上的actor. 如果没有GPU, 
  # 则直接使用ray.remote, 不用带上任何参数以及圆括号.
  @ray.remote(num_gpus=1)
  class NeuralNetOnGPU(object):
      def __init__(self):
          # 设置环境变量告知TensorFlow使用哪些GPU.
          # 注意这些需要在tf.Session被调用之前做.
          os.environ["CUDA_VISIBLE_DEVICES"] = ",".join([str(i) for i in ray.get_gpu_ids()])
          with tf.Graph().as_default():
              with tf.device("/gpu:0"):
                  self.x, self.y_, self.train_step, self.accuracy = construct_network()
                  # 允许在没有任何GPU时在CPU上运行这段代码.
                  config = tf.ConfigProto(allow_soft_placement=True)
                  self.sess = tf.Session(config=config)
                  # 初始化网络.
                  init = tf.global_variables_initializer()
                  self.sess.run(init)

我们可以通过往 ``ray.remote`` 中传入 ``num_gpus=1`` 说明actor需要一块GPU.
注意为了让这样的语句奏效, Ray必须在启动时包含一些GPU, 比如通过 ``ray.init(num_gpus=2)``.
否则当你试图用 ``NeuralNetOnGPU.remote()`` 初始化GPU版actor时, 会抛出一个异常说系统没有足够的GPU.

当创建一个actor时, 它首先会通过 ``ray.get_gpu_ids()`` 获得它被允许使用的GPU列表.
这是一个整数列表, 像 ``[]``, 或者 ``[1]``, 或者 ``[2, 5, 6]``. 因为我们传入了1 ``ray.remote(num_gpus=1)``,
那么将返回一个长度为1的列表.

我们将所有代码整合起来.

.. code-block:: python

  import os
  import ray
  import tensorflow as tf
  from tensorflow.examples.tutorials.mnist import input_data

  ray.init(num_gpus=8)

  def construct_network():
      x = tf.placeholder(tf.float32, [None, 784])
      y_ = tf.placeholder(tf.float32, [None, 10])

      W = tf.Variable(tf.zeros([784, 10]))
      b = tf.Variable(tf.zeros([10]))
      y = tf.nn.softmax(tf.matmul(x, W) + b)

      cross_entropy = tf.reduce_mean(-tf.reduce_sum(y_ * tf.log(y), reduction_indices=[1]))
      train_step = tf.train.GradientDescentOptimizer(0.5).minimize(cross_entropy)
      correct_prediction = tf.equal(tf.argmax(y,1), tf.argmax(y_,1))
      accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))

      return x, y_, train_step, accuracy

  @ray.remote(num_gpus=1)
  class NeuralNetOnGPU(object):
      def __init__(self, mnist_data):
          self.mnist = mnist_data
          # 设置环境变量告知TensorFlow使用哪些GPU.
          # 注意这些需要在tf.Session被调用之前做.
          os.environ["CUDA_VISIBLE_DEVICES"] = ",".join([str(i) for i in ray.get_gpu_ids()])
          with tf.Graph().as_default():
              with tf.device("/gpu:0"):
                  self.x, self.y_, self.train_step, self.accuracy = construct_network()
                  # 允许在没有任何GPU时在CPU上运行这段代码.
                  config = tf.ConfigProto(allow_soft_placement=True)
                  self.sess = tf.Session(config=config)
                  # 初始化网络.
                  init = tf.global_variables_initializer()
                  self.sess.run(init)

      def train(self, num_steps):
          for _ in range(num_steps):
              batch_xs, batch_ys = self.mnist.train.next_batch(100)
              self.sess.run(self.train_step, feed_dict={self.x: batch_xs, self.y_: batch_ys})

      def get_accuracy(self):
          return self.sess.run(self.accuracy, feed_dict={self.x: self.mnist.test.images,
                                                         self.y_: self.mnist.test.labels})


  # 导入MNIST数据集并且告诉Ray如何序列化这些定制化的类.
  mnist = input_data.read_data_sets("MNIST_data", one_hot=True)

  # 创建actor.
  nn = NeuralNetOnGPU.remote(mnist)

  # 运行一段训练并打印准确性.
  nn.train.remote(100)
  accuracy = ray.get(nn.get_accuracy.remote())
  print("Accuracy is {}.".format(accuracy))

传递actor句柄(试验阶段)
-------------------------------------------

Actor句柄可以被传入到其它任务. 作为一个例子, 参见 `异步parameter server例子`_.
为了用一个简单的例子阐明, 考虑一个简单的actor定义.
这部分功能目前处于 **试验阶段** 并且受下面说明的局限性影响.

.. code-block:: python

  @ray.remote
  class Counter(object):
      def __init__(self):
          self.counter = 0

      def inc(self):
          self.counter += 1

      def get_counter(self):
          return self.counter

我们能够定义远端函数(或者actor方法)使用actor句柄.

.. code-block:: python

  @ray.remote
  def f(counter):
      while True:
          counter.inc.remote()

如果我们实例化了一个actor, 我们可以在不同任务间传递它的句柄.

.. code-block:: python

  counter = Counter.remote()

  # 启动一些使用这个actor的任务.
  [f.remote(counter) for _ in range(4)]

  # 打印计数器的值.
  for _ in range(10):
      print(ray.get(counter.get_counter.remote()))

目前actor的局限性
-------------------------

我们目前正在解决如下的一些问题.

1. **actor生命周期管理:** 目前当原始的actor句柄作用范围失效后, 一个任务会被调度到
   actor上用于杀死actor所在进程(新的任务一旦前面的任务都完成后就运行). 当变量作用范围
   失效但是它通过句柄传递给其他任务并且仍在运行时就会出现问题.
2. **返回actor句柄:** actor句柄目前不能从一个远端函数或者actor方法被返回. 类似的,
   ``ray.put`` 不能被一个actor句柄所调用.
3. **被驱逐的actor对象的重新构造:** 如果对一个由actor方法创建的被驱逐的对象调用 ``ray.get``,
   Ray目前将不会重新构造这个对象. 更多信息参见文档 `容错`_.
4. **丢失actor的确定性重构:** 如果一个actor由于节点故障丢失了, 这个actor会在一个
   新节点上以最初的执行顺序进行重新构造. 然而同时新的被调度到这个actor的任务
   可能会在重新执行的任务之间执行. 如果你的应用需要有严格的状态一致性保证则会出现问题.

.. _`异步parameter server例子`: http://ray.readthedocs.io/en/latest/example-parameter-server.html
.. _`容错`: http://ray.readthedocs.io/en/latest/fault-tolerance.html
