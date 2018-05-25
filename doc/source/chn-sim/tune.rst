Ray Tune: 超参数优化框架
===============================================

Ray Tune是一个可扩展的强化学习和深度学习超参数优化框架. 借助高效的搜索算法, 你可以不用修改代码便能将一次试验从单机扩展到一个大型集群运行.

开始入门
---------------

安装
~~~~~~~~~~~~

你需要首先 `安装ray <installation.html>`__ 来导入Ray Tune模块.

快速上手
~~~~~~~~~~~

.. code-block:: python

    import ray
    import ray.tune as tune

    ray.init()
    tune.register_trainable("train_func", train_func)

    tune.run_experiments({
        "my_experiment": {
            "run": "train_func",
            "stop": {"mean_accuracy": 99},
            "config": {
                "lr": tune.grid_search([0.2, 0.4, 0.6]),
                "momentum": tune.grid_search([0.1, 0.2]),
            }
        }
    })


对于你想调优的函数, 加入两行修改(注意我们使用了PyTorch作为一个例子, 但是Ray Tune可以和任意深度学习框架配合使用):

.. code-block:: python
   :emphasize-lines: 1,14

    def train_func(config, reporter):  # 加入一个参数reporter
        model = NeuralNet()
        optimizer = torch.optim.SGD(
            model.parameters(), lr=config["lr"], momentum=config["momentum"])
        dataset = ( ... )

        for idx, (data, target) in enumerate(dataset):
            # ...
            output = model(data)
            loss = F.MSELoss(output, target)
            loss.backward()
            optimizer.step()
            accuracy = eval_accuracy(...)
            reporter(timesteps_total=idx, mean_accuracy=accuracy) # report metrics

这个PyTorch脚本使用Ray Tune对 ``train_func`` 函数运行了一个小规模的网格搜索, 直到停止条件 ``mean_accuracy >= 99`` 满足之前会在命令行中报告状态(对于像_loss_这样随着时间下降的指标, 也可以使用 `neg_mean_loss <https://github.com/ray-project/ray/blob/master/python/ray/tune/result.py#L40>`__ 作为条件):

::

    == Status ==
    Using FIFO scheduling algorithm.
    Resources used: 4/8 CPUs, 0/0 GPUs
    Result logdir: ~/ray_results/my_experiment
     - train_func_0_lr=0.2,momentum=1:  RUNNING [pid=6778], 209 s, 20604 ts, 7.29 acc
     - train_func_1_lr=0.4,momentum=1:  RUNNING [pid=6780], 208 s, 20522 ts, 53.1 acc
     - train_func_2_lr=0.6,momentum=1:  TERMINATED [pid=6789], 21 s, 2190 ts, 100 acc
     - train_func_3_lr=0.2,momentum=2:  RUNNING [pid=6791], 208 s, 41004 ts, 8.37 acc
     - train_func_4_lr=0.4,momentum=2:  RUNNING [pid=6800], 209 s, 41204 ts, 70.1 acc
     - train_func_5_lr=0.6,momentum=2:  TERMINATED [pid=6809], 10 s, 2164 ts, 100 acc

为了报告增量进展, ``train_func`` 周期性调用由Ray Tune传入的 ``reporter`` 函数返回当前时间戳以及在 `ray.tune.result.TrainingResult <https://github.com/ray-project/ray/blob/master/python/ray/tune/result.py>`__ 中定义的其它指标. 增量结果将会被同步写入到集群头节点的本地磁盘中.

了解更多 `关于指定试验 <tune-config.html>`__ .


特性
--------

Ray Tune 具有以下特性:

-  可扩展的实现搜索算法, 例如 `Population Based Training (PBT) <pbt.html>`__, `Median Stopping Rule <hyperband.html#median-stopping-rule>`__, Model-Based Optimization (HyperOpt), and `HyperBand <hyperband.html>`__.

-  集成可视化工具, 包括 `TensorBoard <https://www.tensorflow.org/get_started/summaries_and_tensorboard>`__, `rllab's VisKit <https://media.readthedocs.org/pdf/rllab/latest/rllab.pdf>`__, 及 `parallel coordinates visualization <https://en.wikipedia.org/wiki/Parallel_coordinates>`__.

-  灵活的试验变量生成, 包括grid search, random search, 以及条件参数分布.

-  资源感知的调度, 包括支持并发运行那些可以并行及分布式的算法.


概念
--------

.. image:: tune-api.svg

Ray Tune可以在集群环境调度大量 *试验*. 每个试验运行一个用户定义的Python程序或者对象, 试验通过用户代码传入的一个变量config进行参数化.

为了运行任意给定的函数, 你需要针对一个函数名运行 ``register_trainable``. 这将使所有的Ray worker感知这个函数.

.. autofunction:: ray.tune.register_trainable

Ray Tune提供了一个 ``run_experiments`` 函数生成和运行通过试验信息描述的试验. 所有的试验被一个实现了搜索算法(默认的是FIFO)的 *试验调度器* 调度和管理.

.. autofunction:: ray.tune.run_experiments

Ray Tune可以在Ray能运行的地方使用, 例如在你的笔记本电脑上跑一段有 ``ray.init()`` 嵌入的Python脚本, 或者在一个 `自动扩展集群 <autoscaling.html>`__ 上以获取更大并行性.

你可以 `在这个GitHub地址 <https://github.com/ray-project/ray/tree/master/python/ray/tune>`__ 看到Ray Tune代码.


试验调度器
----------------

默认情况下, Ray Tune使用 ``FIFOScheduler`` 类将试验以串行顺序进行调度. 然而, 你也可以指定一个能够早停试验、扰动参数或者和外部参数服务协作的自定义调度算法. 目前已经实现的调度算法包括
`Population Based Training (PBT) <pbt.html>`__, `Median Stopping Rule <hyperband.html#median-stopping-rule>`__, `Model Based Optimization (HyperOpt) <#hyperopt-integration>`__, 以及 `HyperBand <hyperband.html>`__.

.. code-block:: python

    run_experiments({...}, scheduler=AsyncHyperBandScheduler())


处理大规模数据集
-----------------------

你可能需要在driver上对一个大对象进行计算(例如, 训练数据, 模型权重值)并且在每一次试验中使用这个对象. Ray Tune 提供了一个 ``pin_in_object_store`` 函数可以用来广播这种大对象. 被用这种方式"钉住"的对象将在driver进程被运行时不被替换出Ray的object store, 并且可以被任意任务使用 ``get_pinned_object`` 高效获取.

.. code-block:: python

    import ray
    from ray.tune import register_trainable, run_experiments
    from ray.tune.util import pin_in_object_store, get_pinned_object

    import numpy as np

    ray.init()

    # X_id能在闭包中被引用
    X_id = pin_in_object_store(np.random.random(size=100000000))

    def f(config, reporter):
        X = get_pinned_object(X_id)
        # use X

    register_trainable("f", f)
    run_experiments(...)


HyperOpt集成
--------------------

``HyperOptScheduler`` 是一个由HyperOpt支持用于基于模型的顺序超参数调优的试验调度器.
为了使用这个调度器, 你将需要使用如下命令安装HyperOpt:

.. code-block:: bash

    $ pip install --upgrade git+git://github.com/hyperopt/hyperopt.git

一个关于这个的例子`hyperopt_example.py <https://github.com/ray-project/ray/blob/master/python/ray/tune/examples/hyperopt_example.py>`__.

.. autoclass:: ray.tune.hpo_scheduler.HyperOptScheduler


结果可视化
-------------------

Ray Tune记录每次实验中的试验结果到一个指定目录, 例如, 上例中的~/ray_results/my_experiment. 日志的记录内容与许多可视化工具兼容:

为了在tensorboard中可视化学习过程, 安装TensorFlow:

.. code-block:: bash

    $ pip install tensorflow

接着, 在你开始运行一次实验后, 你可以通过指定结果输出目录的方式使用TensorBoard可视化你的实验. 注意如果你在一个远端集群中运行Ray, 你可以使用 ``ssh -L 6006:localhost:6006 <address>`` 端口转发到你本地机器上tensorboard的端口上:

.. code-block:: bash

    $ tensorboard --logdir=~/ray_results/my_experiment

.. image:: ray-tune-tensorboard.png

为了使用rllab的VisKit(你需要安装一些依赖), 运行如下命令:

.. code-block:: bash

    $ git clone https://github.com/rll/rllab.git
    $ python rllab/rllab/viskit/frontend.py ~/ray_results/my_experiment

.. image:: ray-tune-viskit.png

最后, 为了使用一个 `parallel coordinates visualization <https://en.wikipedia.org/wiki/Parallel_coordinates>`__ 看结果, 打开 `ParallelCoordinatesVisualization.ipynb <https://github.com/ray-project/ray/blob/master/python/ray/tune/ParallelCoordinatesVisualization.ipynb>`__ 并在其cell中运行:

.. code-block:: bash

    $ cd $RAY_HOME/python/ray/tune
    $ jupyter-notebook ParallelCoordinatesVisualization.ipynb

.. image:: ray-tune-parcoords.png


试验检查点
-------------------

为了能够使用检查点, 你必须实现一个Trainable对象(Trainable函数不能够做检查点, 因为它们永远不会将控制返回到它们的调用者). 这样最简单的方式是继承预定义的 ``Trainable`` 类并且实现它的抽象方法 ``_train``, ``_save`` 和 ``_restore``. `(例子)<https://github.com/ray-project/ray/blob/master/python/ray/tune/examples/hyperband_example.py>`__ : 实现这些接口需要支持多路资源的调度算法, 例如HyperBand和PBT.

对于TensorFlow模型训练, 和这个例子很像 `(full tensorflow example) <https://github.com/ray-project/ray/blob/master/python/ray/tune/examples/tune_mnist_ray_hyperband.py>`__:

.. code-block:: python

    class MyClass(Trainable):
        def _setup(self):
            self.saver = tf.train.Saver()
            self.sess = ...
            self.iteration = 0

        def _train(self):
            self.sess.run(...)
            self.iteration += 1

        def _save(self, checkpoint_dir):
            return self.saver.save(
                self.sess, checkpoint_dir + "/save",
                global_step=self.iteration)

        def _restore(self, path):
            return self.saver.restore(self.sess, path)


此外, 检查点可以被用来提供实验容错. 这可以通过设置 ``checkpoint_freq: N`` 和 ``max_failures: M`` 从而每次试验 *N* 次迭代后做检查点并且每次试验最多可以从 *M* 次失败中恢复, 例如:

.. code-block:: python

    run_experiments({
        "my_experiment": {
            ...
            "checkpoint_freq": 10,
            "max_failures": 5,
        },
    })

如下对象接口必须要实现用于检查点使用:

.. autoclass:: ray.tune.trainable.Trainable


客户端API
----------

你可以使用Tune Client API给一个正在运行的实验增加或删除试验. 这需要你验证下是否安装了 ``requests`` 库:

.. code-block:: bash

    $ pip install requests

为了使用客户端API, 你可以通过 ``with_server=True`` 启动你的实验:

.. code-block:: python

    run_experiments({...}, with_server=True, server_port=4321)

接着在客户端你可以使用下述对象. 服务端地址默认为 ``localhost:4321``. 在集群环境你可能需要重定向端口 (例如 ``ssh -L <local_port>:localhost:<remote_port> <address>``) 以便你可以在自己本地机器上使用Client.

.. autoclass:: ray.tune.web_server.TuneClient
    :members:


客户端API例子, 参见 `Client API Example <https://github.com/ray-project/ray/tree/master/python/ray/tune/TuneClient.ipynb>`__.


例子
--------

你可以在这里找到一个示例列表 `使用Ray Tune以及它的不同特性 <https://github.com/ray-project/ray/tree/master/python/ray/tune/examples>`__.
