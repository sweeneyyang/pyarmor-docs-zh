安装和卸载
==========

|PyArmor| 可以直接从这里 PyPi_ 下载, 但是更方便的方式是通过 pip_ ，直接
运行下面的命令进行安装::

    pip install pyarmor

需要升级的话，执行下面的命令::

    pip install --upgrade pyarmor

另外还有一个图形界面的包，需要的话可以安装::

    pip install pyarmor-webui

一旦成功安装，就可以直接运行命令 ``pyarmor`` ，例如::

    pyarmor --version

这个命令会显示版本信息 ``PyArmor Version X.Y.Z`` 或者 ``PyArmor Trial Version X.Y.Z``.

如果命令无法执行，请检查环境变量中是否包含可执行文件所在的路径。

可用的命令
----------

使用 `pip` 安装之后，有两个可用的命令：

* ``pyarmor`` 这是主要的工具，参考 :ref:`使用 PyArmor`.

* ``pyarmor-webui`` 用来打开网页版的可视化界面

如果没有使用 `pip` 进行安装，上述命令无法使用，需要运行 `Python` 来执行
相应的脚本。 ``pyarmor`` 等价于执行 :file:`{pyarmor-folder}/pyarmor.py`,
``pyarmor-webui`` 等价于执行 :file:`{pyarmor-folder}/pyarmor/webui/server.py`

.. _完全卸载:

完全卸载
--------

下列文件可能会在 `pyarmor` 运行时被创建::

    ~/.pyarmor/.pyarmor_capsule.zip         (从 v6.2.0 开始)
    ~/.pyarmor/license.lic                  (从 v5.8.0 开始)
    ~/.pyarmor/platforms/

    {pyarmor-folder}/license.lic            (在 v5.8.0 之前)
    ~/.pyarmor_capsule.zip                  (在 v6.2.0 之前)

    /path/to/project/.pyarmor_config        (如果使用了工程）

执行下面的命令进行完全卸载::

    pip uninstall pyarmor
    rm -rf ~/.pyarmor

    rm -rf {pyarmor-folder}                 (在 v5.8.0 之前)
    rm -rf ~/.pyarmor_capsule.zip           (在 v6.2.0 之前)

    rm /path/to/project/.pyarmor_config

.. note::

   使用不同用户登陆，路径 ``~`` 可能不一样。

.. include:: _common_definitions.txt
