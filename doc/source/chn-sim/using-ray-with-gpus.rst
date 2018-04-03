基于GPU使用Ray
===================

GPU在机器学习应用中非常关键. Ray可以让远端函数和actor在 ``ray.remote`` 装饰器中指定它们对GPU的需求.

带GPU启动Ray
----------------------

为了让远端函数和actor使用GPU, Ray必须知道有多少GPU可用. 如果你在单机上启动Ray, 你可以通过如下方式指定GPU的数量.

.. code-block:: python

  ray.init(num_gpus=4)

如果你没有传递 ``num_gpus`` 参数, Ray将会假定机器上没有GPU.

如果你使用 ``ray start`` 命令启动Ray, 你可以通过 ``--num-gpus`` 参数的方式指定机器上的GPU数量.

.. code-block:: bash

  ray start --head --num-gpus=4

**注意:** 没什么会阻止你传入一个比机器上GPU数量大的 ``num_gpus`` 值.
在这种情况下, Ray会在需要GPU的任务调度时假定机器上有你指定数目的GPU.
当分配到不存在GPU上的任务试图使用GPU时则会发生问题.

基于GPU使用远端函数
--------------------------------

如果远端函数需要GPU, 则在远端函数装饰器中指明需要的GPU数.

.. code-block:: python

  @ray.remote(num_gpus=1)
  def gpu_method():
      return "This function is allowed to use GPUs {}.".format(ray.get_gpu_ids())

在远端函数内部, 对 ``ray.get_gpu_ids()`` 的调用将会返回一个整数列表代表远端函数被允许使用哪些GPU.

**注意:** 如上定义的函数 ``gpu_method`` 并没有实际使用任何GPU.
Ray将会将这个远端函数调度到一台至少拥有一块GPU的机器上, 并且当其运行时为其保留一块GPU,
然而实际对GPU的利用取决于这个函数本身.
这通常由像TensorFlow之类的外部库来完成. 这里有一个实际使用GPU的例子.
注意为了能够运行这个例子, 你需要安装GPU版本的TensorFlow.

.. code-block:: python

  import os
  import tensorflow as tf

  @ray.remote(num_gpus=1)
  def gpu_method():
      os.environ["CUDA_VISIBLE_DEVICES"] = ",".join(map(str, ray.get_gpu_ids()))
      # 创建一个TensorFlow session.
      # TensorFlow会限制它自身使用由环境变量CUDA_VISIBLE_DEVICES指定的GPU.
      tf.Session()

**注意:** 当实现 ``gpu_method`` 时忽略 ``ray.get_gpu_ids`` 也是可能的, 这样会使用机器上的所有GPU.
Ray不会去避免这种情况发生, 这将导致许多worker在同一时间使用同一块GPU.
举个例子, 如果 ``CUDA_VISIBLE_DEVICES`` 环境变量没有被设置, 那么TensorFlow会试图使用机器上的所有GPU.

基于GPU使用actor
----------------------

当定义一个使用GPU的actor时, 在 ``ray.remote`` 装饰器中指明actor实例需要的GPU的数量.

.. code-block:: python

  @ray.remote(num_gpus=1)
  class GPUActor(object):
      def __init__(self):
          return "This actor is allowed to use GPUs {}.".format(ray.get_gpu_ids())

当actor被创建后, 这些GPU将会在actor的生命周期中为这个actor保留.

注意Ray在启动的时候指明的GPU数量需要不比你传入 ``ray.remote`` 装饰器的数字小.
否则如果你传入一个数大于你在 ``ray.init`` 时传入的数, 当actor实例化的时候将会抛出一个异常.

下面的代码示例说明如何在一个actor中通过TensorFlow使用GPU.

.. code-block:: python

  @ray.remote(num_gpus=1)
  class GPUActor(object):
      def __init__(self):
          self.gpu_ids = ray.get_gpu_ids()
          os.environ["CUDA_VISIBLE_DEVICES"] = ",".join(map(str, self.gpu_ids))
          # TensorFlow会限制它自身使用由环境变量CUDA_VISIBLE_DEVICES指定的GPU.
          self.sess = tf.Session()

问题定位
---------------

**注意:** 目前, 当一个worker执行一个使用GPU的任务时,
这个任务将会申请GPU上的显存并且可能在任务执行完之后也不会释放. 这会造成一些问题. 参见 `这个issue`_.

.. _`这个issue`: https://github.com/ray-project/ray/issues/616
