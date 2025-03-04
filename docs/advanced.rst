.. _advanced topics:

.. _高级用法:

高级用法
========

.. _using super mode:

.. _使用超级模式加密脚本:

使用超级模式加密脚本
--------------------

:ref:`超级模式` 是 6.2.0 的新功能，超级模式只需要一个运行辅助文件，没有了所谓的
引导代码，所有的加密脚本都长得一个样，大大简化了加密脚本的使用方法，并且极大的提
高了安全性，唯一的缺点是只支持最新版本的Python，包括 2.7，3.7，3.8，3.9。

超级模式在命令行使用 ``--advanced 2`` 来指定，例如::

  pyarmor obfuscate --advanced 2 foo.py

而发布加密脚本的时候，只需要把扩展模块 :mod:`pytransform` ，放在任意的 Python 路径
下面，加密后的脚本就可以正常运行。

如果需要对加密脚本进行限制，那么使用首先生成一个包含约束的 ``license.lic`` ，例如，
绑定到网卡::

  pyarmor licenses --bind-mac xx:xx:xx:xx regcode-01

然后在加密脚本的时候使用 ``--with-license`` 指定这个文件，例如::

  pyarmor obufscate --with-license licenses/regcode-01/license.lic \
                    --advanced 2 foo.py

这样就可以把指定的许可文件嵌入到扩展模块里面，如果不想把许可文件嵌入到扩展模块里
面，而是使用外部 ``license.lic`` ，这样可以方便替换许可文件，那么在加密的
时候需要指定 ``--with-license outer`` ，例如::

  pyarmor obfuscate --with-license --advanced 2 outer foo.py

更多说明，请参考下一个章节。

.. _如何使用外部许可文件:

如何使用外部许可文件
--------------------

从 6.3.0 开始，默认情况下许可证是被嵌入到动态库中，如果不想把许可文件嵌入到扩展
模块里面，而是使用外部 ``license.lic`` ，这样可以方便替换许可文件。那么在加密的
时候需要指定 ``--with-license outer`` ，例如::

  pyarmor obfuscate --with-license outer foo.py

这样，运行加密脚本的时候就会在外部查找许可文件，查找外部 ``license.lic`` 的顺序

#. 如果设定了环境变量 ``PYARMOR_LICENSE`` ，直接使用这里指定的文件名
#. 如果设置了 ``sys.PYARMOR_LICENSE`` ，直接使用这里指定的文件名
#. 如果没有，那么查找当前路径下面的 ``license.lic``
#. 如果没有，对于非超级模式，还会在运行辅助包 ``pytransform`` 所在的路径查找 ``license.lic``
#. 没有找到就报错

例如，使用 ``sys.PYARMOR_LICENSE`` 来指定外部许可证

对于非超级模式，可以直接修改运行辅助文件 ``dist/pytransform/__init__.py`` 中的函
数 ``pyarmor_runtime`` ，在其中增加一行::

    sys.PYARMOR_LICENSE = '/path/to/license.lic'

对于超级模式，需要把扩展模块转换成为包，例如，转换扩展模块 `pytransform.so` 为包
`pytransform`::

    cd dist
    mkdir pytransform
    mv pytransform.so pytransform/

新增 ``dist/pytransform/__init__.py``

.. code:: python

    import sys
    sys.PYARMOR_LICENSE = '/path/to/license.lic'
    name = 'pytransform'
    m = __import__(name, globals(), locals(), ['*'])
    sys.modules[__name__].__dict__.update(m.__dict__)

.. _obfuscating many packages:

加密和使用多个包
----------------

假定有三个包 `pkg1`, `pkg2`, `pkg2` 需要加密，使用公共的运行辅助文件，
然后可以从其他脚本导入这些加密的包。

首先切换到工作路径，创建三个工程::

    mkdir build
    cd build

    pyarmor init --src /path/to/pkg1 --entry __init__.py pkg1
    pyarmor init --src /path/to/pkg2 --entry __init__.py pkg2
    pyarmor init --src /path/to/pkg3 --entry __init__.py pkg3

生成公共的 :ref:`运行辅助包` ，保存在 `dist` 目录下面::

    pyarmor build --output dist --only-runtime pkg1

或者也可以使用命令 :ref:`runtime` 直接生成 :ref:`运行辅助包`::

    pyarmor runtime --output dist

在分别加密三个包，也保存到 `dist` 下面::

    pyarmor build --output dist --no-runtime pkg1
    pyarmor build --output dist --no-runtime pkg2
    pyarmor build --output dist --no-runtime pkg3

查看并使用加密的包::

    ls dist/

    cd dist
    python -c 'import pkg1
    import pkg2
    import pkg3'

.. note::

   输出目录 `dist` 下面的运行辅助包 :mod:`pytransform` 可以被拷贝到任何的 Python
   可以导入的目录下面。

   从 v5.7.2 之后，运行辅助包也可以使用命令 :ref:`runtime` 单独生成::

       pyarmor runtime

.. _solve conflicts with other obfuscated libraries:

.. _如何加密能和其他加密包共存的包:

如何加密能和其他加密包共存的包
------------------------------

.. note:: New in v5.8.7

假设有两个包分别被两个不同的开发人员进行加密，那么这两个包能不能在同一个 Python
解释器中运行呢？

如果这两个包都是被试用版加密，那么没有问题。但是如果任何一个是被注册版本的
PyArmor 加密，那么答案是否定的。

从 v5.8.7 开始，使用选项 ``--enable-suffix`` 来加密时， :ref:`运行辅助包` 的名称
不在固定为 ``pytransform`` ，而是会有一个唯一性的后缀，这样不同的加密包就可以实
现共存。例如::

    pyarmor obfuscate --enable-suffix foo.py

加密后的输出目录结构如下::

    dist/
        foo.py
        pytransform_vax_000001/
            __init__.py
            ...

其中后缀 ``_vax_000001`` 是基于 PyArmor 的注册码生成的具有唯一性的字符串。

对于使用工程加密的方式，则需要使用命令 :ref:`config` 来设置::

    pyarmor config --enable-suffix 1
    pyarmor build -B

使用下面的方式可以禁用后缀模式::

    pyarmor config --enable-suffix 0
    pyarmor build -B

.. _distributing obfuscated packages:

发布多个加密包
--------------

如果有多个加密的包需要发布，推荐的方式是启用选项 ``--enable-suffix`` 生成一个单
独的 :ref:`运行辅助包` ，然后其他加密包共享这个运行模块。

例如，首先使用命令 :ref:`runtime` 生成 :ref:`运行辅助包`::

    pyarmor runtime --enable-suffix -O dist/shared

输出的包的名字可能是 ``dist/shared/pytransform_vax_000001`` ，不同的用户包的名字
可能不一样。

对每一个包，使用下面的方式进行加密::

    pyarmor obfuscate --enable-suffix --recursive --bootstrap 2 \
                      -O dist/pkg1 --runtime @dist/shared src/pkg1/__init__.py

如果选项 ``--runtime`` 不可用，它是在 v6.3.7 才有的，那么直接使用选项
``--no-runtime``::

    pyarmor obfuscate --enable-suffix --recursive --bootstrap 2 \
                      -O dist/pkg1 --no-runtime src/pkg1/__init__.py

接下来把运行辅助包 `pytransform_vax_000001` 单独发布成为一个独立的包。

最后，发布加密的包 `dist/pkg1` ，在安装脚本 ``setup.py`` 里面增加一个依赖包::

    install_requires=['pytransform_vax_000001']

对于其他包 `pkg2` ， `pkg3` 等，和 `pkg1` 一样进行发布。

.. _distributing obfuscated scripts to other platform:

.. _跨平台发布加密脚本:

跨平台发布加密脚本
------------------

因为加密脚本的运行文件中有平台相关的动态库，所以跨平台发布需要指定目标平台。

首先使用命令 :ref:`download` 列出所有支持的标准平台名称::

    pyarmor download
    pyarmor download --help-platform

使用选项 ``--list`` 会显示详细的动态库特征信息::

    pyarmor download --list
    pyarmor download --list windows
    pyarmor download --list windows.x86_64

然后在加密脚本的时候指定目标平台名称::

    pyarmor obfuscate --platform linux.armv7 foo.py

    # For project
    pyarmor build --platform linux.armv7

.. _obfuscating scripts with different features:

.. _使用不同特征的动态库:

使用不同特征的动态库
~~~~~~~~~~~~~~~~~~~~

同一个平台下面，包含多个可用的动态库，不同的动态库具备不同的特征。例如，在相同的
平台 ``windwos.x86_64`` 下面，有两个动态库 ``windows.x86_64.0`` 和
``windows.x86_64.7`` ，其中最后的数字表示动态库的特征:

  - 0: 没有反调试、动态代码、高级模式等特征，速度最快
  - 7: 包含反调试、动态代码、高级模式等特征，安全性最高

如果不指定特征码，默认是选择安全性最高的动态库，有些不平台只支持部分特征的动态库。
在跨平台加密的时候也可以指定平台的特征，例如::

    pyarmor obfuscate --platform linux.x86_64.7 foo.py

需要注意的是不同特征的动态库相互并不兼容，例如，默认情况下 Windows 平台下使用的
是高特征的动态库，当使用下面的命令在 Windows 平台下面加密低特征的动态库的时候::

    pyarmor obfuscate --platform linux.arm.0 foo.py

在控制台能看到 PyArmor 会自动重启，使用低特征的库加载之后才对脚本进行加密。

也可以使用环境变量 ``PYARMOR_PLATFORM`` 直接设置当前平台使用的动态库的特征。例如::

    PYARMOR_PLATFORM=windows.x86_64.0 pyarmor obfuscate --platform linux.arm.0 foo.py

    # 在 Windows 平台
    set PYARMOR_PLATFORM=windows.x86_64.0
    pyarmor obfuscate --platform linux.arm.0 foo.py
    set PYARMOR_PLATFORM=

.. _running obfuscated scripts in multiple platforms:

.. _让加密脚本可以在多个平台运行:

让加密脚本可以在多个平台运行
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

从 v5.7.5 版本开始，平台名称已经标准化，所有可用名称在这里 :ref:`标准平台名称`
，并且支持运行加密脚本在多个平台。

为了支持加密脚本在多个平台运行，需要把相关平台的动态库都添加到 :ref:`运行辅助包`
中，这样就可以在这些平台正常运行加密脚本。例如，使用下面的命令可以加密一个可运行
于 Windows/Linux/MacOS 下面的脚本::

    pyarmor obfuscate --platform windows.x86_64 \
                      --platform linux.x86_64 \
                      --platform darwin.x86_64 \
                      foo.py

也可以使用命令 :ref:`runtime` 单独生成可以运行多个平台的 `运行辅助包` ，这样就不
需要每次加密的时候都生成这些辅助文件。例如::

    pyarmor runtime --platform windows.x86_64,linux.x86_64,darwin.x86_64
    pyarmor obfuscate --no-runtime --recursive \
                      --platform windows.x86_64,linux.x86_64,darwin.x86_64 \
                      foo.py

即便使用了 ``--no-runtime`` ，在加密脚本的时候也需要指定运行的平台，因为加密脚本
会在启动的时候检查动态库，只有指定的动态库才能通过检查。如果指定了选项
``--no-cross-protection`` ，加密脚本就不会在检查动态库，那么加密的时候就不需要指
定运行平台，例如::

    pyarmor obfuscate --no-runtime --recursive --no-cross-protection foo.py

.. note::

   如果指定了平台的特征，例如 ``windows.x86_64.7`` ，那么需要注意的是所有的平台
   必须具备相同的特征，不同特征的动态库是无法共存在同一个包里面的。

.. note::

   如果加密后的脚本无法运行，可以尝试使用下面的命令升级已经下载的动态库::

       pyarmor download --update

   也可以直接删除缓存的动态库目录，默认是 ``$HOME/.pyarmor/platforms``

.. _obfuscating scripts by other python version:

使用不同版本 Python 加密脚本
----------------------------

如果装了多个版本的 Python ，那么使用 `pip` 安装的 `pyarmor` 使用的是默
认的 Python 版本。如果需要使用其他版本的 Python 来加密脚本，需要显示指
定 Python 解释器。

例如，首先找到 :file:`pyarmor.py` 的位置::

      find /usr/local/lib -name pyarmor.py

通常在大多数 linux 系统，它会在 `/usr/local/lib/python2.7/dist-packages/pyarmor`

然后使用下面的方式运行::

    /usr/bin/python3.6 /usr/local/lib/python2.7/dist-packages/pyarmor/pyarmor.py

也可以创建一个便捷脚本 `/usr/local/bin/pyarmor3` ，内容如下::

    /usr/bin/python3.6 /usr/local/lib/python2.7/dist-packages/pyarmor/pyarmor.py "$@"

赋予其执行权限::

    chmod +x /usr/local/bin/pyarmor3

然后就可以直接使用 `pyarmor3`

在 Windows 下面就需要创建一个批处理 `pyarmor3.bat` ，内容如下::

    C:\Python36\python C:\Python27\Lib\site-packages\pyarmor\pyarmor.py %*

.. _run bootstrap code in plain scripts:

.. _在没有加密的脚本中运行引导代码:

在没有加密的脚本中运行引导代码
------------------------------

有时候需要在普通脚本中运行引导代码，这样就可以正常的导入其他加密脚本，
而不需要在每一个加密脚本中到插入引导代码。在 v5.7.0 之前，可以直接把两
行 :ref:`引导代码` 插入到普通脚本中，但是之后的版本，为了提高安全性，
已经不允许在普通脚本中直接运行引导代码，必须使用一种折衷的方式。

首先使用命令 :ref:`runtime` 生成一个引导辅助包 :mod:`pytransform_bootstrap`::

    pyarmor runtime -i

然后把生成的辅助包拷贝到脚本所在的目录::

    mv dist/pytransform_bootstrap /path/to/script

也可以把这个包拷贝到 Python 的库目录，例如::

    mv dist/pytransform_bootstrap /usr/lib/python3.5/ (For Linux)
    mv dist/pytransform_bootstrap C:/Python35/Lib/ (For Windows)

最后修改普通脚本，在其中插入一条语句::

    import pytransform_bootstrap

这样就可以在其后导入并使用其他加密模块。

.. note::

   在 v5.8.1 之前，需要使用人工的方式生成这个引导辅助包::

    echo "" > __init__.py
    pyarmor obfuscate -O dist/pytransform_bootstrap --exact __init__.py

下面是一个实际的例子，运行加密脚本的单元测试用例。

.. _run unittest of obfuscated scripts:

运行加密脚本的单元测试
~~~~~~~~~~~~~~~~~~~~~~

因为大部分的加密脚本都没有 :ref:`引导代码` ，所以在运行单元测试之前，必须首先运
行引导代码。

假设单元测试脚本为 :file:`/path/to/tests/test_foo.py` ，那么首先修改这个单元测试
脚本，参考 `在没有加密的脚本中运行引导代码`_

这样，这个测试脚本就可以导入被加密的模块进行正常的测试::

    cd /path/to/tests
    python test_foo.py

还有一种方式就是直接修改系统包 :mod:`unittest` ，首先要把引导辅助包拷贝到 Python
的系统库路径下面，参考 `在没有加密的脚本中运行引导代码`_

然后在修改 :file:`/path/to/unittest/__init__.py` ，插入语句::

    import pytransform_bootstrap

这样，所有的单元测试脚本就都可以直接来测试加密后的模块了。如果有很多单
元测试脚本，这种方式会更方便一些。

.. _let python interpreter recognize obfuscated scripts automatically:

让 Python 自动识别加密脚本
--------------------------

下面有几种情况可能会需要让 Python 自动识别加密脚本:

* 几乎所有的脚本都会被作为主脚本来运行
* 在加密脚本中使用模块 `multiprocessing` 创建新进程
* 使用到 `Popen` 或者 `os.exec` 等调用加密后的脚本
* 其他任何需要在很多脚本里面插入引导代码的情况

一种解决方案就是为每一个相关的加密脚本添加引导代码，但是这会有些麻烦。
另外一种比较简单的解决方案就是让 Python 能够自动识别加密脚本，这样任何
一个加密脚本不需要引导代码就可以正常运行。

下面是基本操作步骤:

1. 首先生成引导辅助包 :mod:`pytransform_bootstrap`::

    pyarmor runtime -i

   在 v5.8.1 之前，需要通过加密一个空脚本的方式生成引导辅助包::

    echo "" > __init__.py
    pyarmor obfuscate -O dist/pytransform_bootstrap --exact __init__.py

2. 其次需要建立一个运行加密脚本的虚拟环境，把引导辅助包拷贝到虚拟环境
   的库路径，例如::

    # For windows
    mv dist/pytransform_bootstrap venv/Lib/

    # For linux
    mv dist/pytransform_bootstrap venv/lib/python3.5/

4. 最后修改 ``venv/lib/pythonX.Y/site.py`` 或者 ``venv/lib/site.py`` ,
   插入一条导入语句::

    import pytransform_bootstrap

    if __name__ == '__main__':
        ...

也可以把这行代码添加到 ``main`` 函数里面，总之，只要能得到执行就可以。

这样就可以使用虚拟环境中 ``python`` 直接运行加密脚本了。 这主要使用到
了 Python 在启动过程中默认会自动导入模块 ``site`` 的特性来实现，参考

https://docs.python.org/3/library/site.html

.. note::

    这里配置的是运行加密脚本的环境，在这里 `pyarmor` 是无法运行的。

.. note::

    在 v5.7.0 之前，需要根据 :ref:`运行辅助文件` 人工创建引导辅助包

.. _obfuscating python scripts in different modes:

使用不同的模式来加密脚本
------------------------

:ref:`高级模式` 是从 PyArmor 5.5.0 引入的新特性，默认情况下是没有启用的。如果需
要使用高级模式来加密脚本，额外指定选项 ``--advanced``::

    pyarmor obfuscate --advanced 1 foo.py

从 PyArmor 5.2 开始, :ref:`约束模式` 是默认设置。

使用选项 ``--restrict`` 指定其他约束模式，例如::

    pyarmor obfuscate --restrict=2 foo.py
    pyarmor obfuscate --restrict=3 foo.py

    # For project
    cd /path/to/project
    pyarmor config --restrict 4
    pyarmor build -B

如果需要禁用各种约束，那么使用下面的命令加密脚本::

    pyarmor obfuscate --restrict=0 foo.py

    # For project
    pyarmor config --restrict=0
    pyarmor build -B

如果使用了命令 :ref:`licenses` 创建了自定义的许可证，那么禁用各种约束
需要在生成许可证的时候使用选项 ``--disable-restrict-mode`` ，例如::

    pyarmor licenses --disable-restrict-mode r001

    pyarmor obfuscate --with-license=licenses/r001/license.lic foo.py

    # For project
    pyarmor config --with-license=licenses/r001/license.lic
    pyarmor build -B

指定 :ref:`代码加密模式`, :ref:`代码包裹模式`, :ref:`模块加密模式` 需
要 :ref:`使用工程` 来加密脚本，直接使用命令 :ref:`obfuscate` 无法改变
这些加密模式。例如::

    pyarmor init --src=src --entry=main.py .
    pyarmor config --obf-mod=1 --obf-code=1 --wrap-mode=0
    pyarmor build

.. _using plugin to extend license type:

.. _使用插件扩展认证方式:

使用插件扩展认证方式
--------------------

PyArmor 可以通过插件来扩展加密脚本的认证方式，例如检查网络时间而不是本地时间来校
验有效期。

首先定义插件文件 `check_ntp_time.py
<https://github.com/dashingsoft/pyarmor/blob/master/plugins/check_ntp_time.py>`_
插件的主函数是 `check_ntp_time` ，另外一个重要的函数是 `_get_license_data` ，用
来从加密脚本的 `license.lic` 许可文件中获取自定义的数据信息。

然后在主脚本 `foo.py
<https://github.com/dashingsoft/pyarmor/blob/master/plugins/foo.py>`_ 插入下列两
行注释::

    # {PyArmor Plugins}
    # PyArmor Plugin: check_ntp_time()

执行下面的命令进行加密::

    pyarmor obfuscate --plugin check_ntp_time foo.py

插件对应的文件一般存放在当前目录，如果存放在其他目录的话，可以指定绝对路径，例如::

    pyarmor obfuscate --plugin /usr/share/pyarmor/check_ntp_time foo.py

最后为加密脚本生成许可文件，使用 ``-x`` 把自定义的有效期存储到认证文件，这样就可
以通过插件脚本中定义的函数 `_get_license_data` 来读取这个数据::

    pyarmor licenses -x 20190501 rcode-001
    cp licenses/rcode-001/license.lic dist/

更多插件实例，参考 https://github.com/dashingsoft/pyarmor/tree/master/plugins

关于插件的工作原理，参考 :ref:`如何处理插件`

.. _bundle obfuscated scripts to one executable file:

.. _打包加密脚本成为一个单独的可执行文件:

打包加密脚本成为一个单独的可执行文件
------------------------------------

使用下面的命令可以把脚本 `foo.py` 加密之后并打包成为一个单独的可执行文件::

    pyarmor pack -e " --onefile" foo.py

其中 ``--onefile`` 是 `PyInstaller`_ 的选项，使用 ``-e`` 可以传递任何
`Pyinstaller`_ 支持的选项，例如，指定可执行文件的图标::

    pyarmor pack -e " --onefile --icon logo.ico" foo.py

如果不想把加密脚本的许可文件 ``license.lic`` 打包到可执行文件，而是和可执行文件
放在一起，这样方便为不同的用户生成不同的许可文件。那么需要使用 `PyInstaller`_ 提
供的 ``--runtime-hook`` 功能在加密脚本运行之前把许可文件拷贝到指定目录，下面是具
体的操作步骤：

1. 新建一个文件 `copy_license.py`::

    import sys
    from os.path import join, dirname
    with open(join(dirname(sys.executable), 'license.lic'), 'rb') as src:
        with open(join(sys._MEIPASS, 'license.lic'), 'wb') as dst:
            dst.write(src.read())

2. 运行下面的命令打包加密脚本::

    pyarmor pack --clean --without-license -x " --exclude copy_license.py" \
            -e " --onefile --icon logo.ico --runtime-hook copy_license.py" foo.py

   选项 ``--without-license`` 告诉 :ref:`pack` 不要把加密脚本的许可文件打包进去，
   使用 `PyInstaller`_ 的选项 ``--runtime-hook`` 可以让打包好的可执行文件，在启动
   的时候首先去调用 :file:`copy_licesen.py` ，把许可文件拷贝到相应的目录。

   命令执行成功之后，会生成一个打包好的文件 :file:`dist/foo.exe`

   尝试运行这个可执行文件，应该会报错。

3. 使用命令 :ref:`licenses` 生成新的许可文件，并拷贝到 :file:`dist/` 下面::

    pyarmor licenses -e 2020-01-01 tom
    cp license/tom/license.lic dist/

4. 这时候在双击运行 :file:`dist/foo.exe` ，在 2020-01-01 之前应该就可以正常运行

.. _bundle obfuscated scripts with customized spec file:

.. _使用定制的 .spec 文件打包加密脚本:

使用定制的 .spec 文件打包加密脚本
---------------------------------

如果已经有写好的 `.spec` 文件能够成功打包，例如::

    pyinstaller myscript.spec

那么可以首先打包好脚本，然后参考 `使用加密脚本直接替换PyInstaller生成的可执行文件`_

也可以使用 :ref:`pack` 和选项 ``-s`` 直接加密并打包 `myscript.py`::

    pyarmor pack -s myscript.spec myscript.py

如果打包的时候出现下面的错误::

    Unsupport .spec file, no XXX found

那么请检查 `.spec` 文件，确保下列两行在文件中存在（没有缩进）::

    a = Analysis(...

    pyz = PYZ(...

并且在创建 `Analysis` 对象的时候下列三个命名参数也存在，例如::

    a = Analysis(
        ...
        pathex=...,
        hiddenimports=...,
        hookspath=...,
        ...
    )

PyArmor 会自动把需要的相关参数添加到这些行。但是在 v5.9.6 之前，需要人工进行添加:

* 增加模块 ``pytransform`` 到 `hiddenimports`
* 增加额外的路径 ``DISTPATH/obf/temp`` 到 `pathex` 和 `hookspath`

修改后的文件大概会是这样子的::

    a = Analysis(['myscript.py'],
                 pathex=[os.path.join(DISTPATH, 'obf', 'temp'), ...],
                 binaries=[],
                 datas=[],
                 hiddenimports=['pytransform', ...],
                 hookspath=[os.path.join(DISTPATH, 'obf', 'temp'), ...],
                 ...

.. note::

   这个功能是在 v5.8.0 新增加的

   在 v5.8.2 之前，额外的路径是 ``DISTPATH/obf`` 而不是 ``DISTPATH/obf/temp``

.. _improving the security by restrict mode:

.. _使用约束模式增加加密脚本安全性:

使用约束模式增加加密脚本安全性
------------------------------

默认约束模式仅限制不能修改加密脚本，为了提高安全性，可以使用约束模式 2 来加密
Python 应用程序，例如::

    pyarmor obfuscate --restrict 2 foo.py

约束模式 2 不允许从没有加密的脚本中导入加密的脚本，从而更高程度的保护了加密脚本
的安全性。

如果对安全性要求更高，可以使用约束模式 3 ，例如::

    pyarmor obfuscate --restrict 3 foo.py

约束模式 3 会检查每一个加密函数的调用，不允许加密的函数被非加密的脚本调用。

上述两种模式并不适用于 Python 包的加密，因为对于 Python 包来说，必须允许加密的脚
本被其他非加密的脚本导入和调用。为了提高 Python 包的安全性，可以采取下面的方案:

* 把需要供外部使用的函数集中到包的某一个或者几个文件
* 使用约束模式 1 加密这些需要被外部调用的文件
* 使用约束模式 4 加密其他的脚本文件

例如，包 `mypkg` 有两个文件:

* __init__.py
* foo.py

其中 `mypkg/__init__.py` 内容如下

.. code-block:: python

    from .foo import hello

    def open_hello(msg):
        print('This is public hello: %s' % msg)

    def proxy_hello(msg):
        print('This is proxy hello from foo: %s' % msg)
        hello(msg)

现在使用下面的方式来加密这个包::

    cd /path/to/mypkg
    pyarmor obfuscate -O obf/mypkg --exact __init__.py
    pyarmor obfuscate -O obf/mypkg --restrict 4 --recursive --exclude __init__.py .

那么，对于加密后的包，可以直接调用 `__init__.py` 中任何函数::

    cd /path/to/mypkg/obf
    python

    >>> import mypkg
    >>> mypkg.open_hello("it should work")
    >>> mypkg.proxy_hello("also OK")

但是却不能直接调用 `mypkg.foo` 中任何函数。例如::

    cd /path/to/mypkg/obf
    python

    >>> import mypkg
    >>> mypkg.foo.hello("it should not work")

关于约束模式的详细说明，请参考 :ref:`约束模式`

.. _使用插件来进一步提高安全性:

使用插件来进一步提高安全性
--------------------------

使用插件可以自由的加入自己的私有检查代码到被加密后的脚本，但是又不影响原来脚本的
执行。因为这些代码一般情况下在没有加密的脚本中是无法正常运行的，如果直接把这些代
码写到脚本里，原来的脚本就无法正常调试。

通过增加私有的检查代码，可以很大程度上提高加密脚本的安全性，因为没有人知道你的检
查逻辑，并且你可以随时更改这些检查逻辑。例如检查当前是否有调试器进程，检查调用者
的代码和 `sum(co_code)` ，当前函数的调用者可以通过 `sys._getframe` 得到等等。

使用内联插件检查动态库没有被修改
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

虽然 `PyArmor` 自身提供了交叉保护功能来保护动态库，但是还可以增加自己的私有检查
点。下面这个例子使用内联插件来再次检查动态库的修改时间，首先在 ``main.py`` 中增
加下面的注释

.. code:: python

  # PyArmor Plugin: import os
  # PyArmor Plugin: libname = os.path.join( os.path.dirname( __file__ ), '_pytransform.so' )
  # PyArmor Plugin: if not os.stat( libname ).st_mtime_ns == 102839284238:
  # PyArmor Plugin:     raise RuntimeError('Invalid Library')

然后调用下面的加密命令来启用这个内联插件::

  pyarmor obfuscate --plugin on main.py

这样，加密脚本在运行的时候就会首先运行下面的插件代码

.. code:: python

  import os
  libname = os.path.join( os.path.dirname( __file__ ), '_pytransform.so' )
  if not os.stat( libname ).st_mtime_ns == 102839284238:
      raise RuntimeError('Invalid Library')

.. _checking imported function is obfuscated:

使用插件检查被调用的函数是否经过加密
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

运行辅助模块 :mod:`pytransform` 里面的函数 :func:`check_armored` 和修饰函数
:func:`assert_armored` 可以用来检查指定的函数是否是加密函数，从而避免其他人调用自
己的关键函数。

假设主脚本为 `main.py`, 需要调用模块 `foo.py` 里面的方法 `connect`, 并且需要传递
敏感数据作为参数。脚本的相关代码如下

.. code:: python

    #
    # This is main.py
    #

    import foo

    def start_server():
        foo.connect('root', 'root password')
        foo.connect2('user', 'user password')

    #
    # This is foo.py
    #

    def connect(username, password):
        mysql.dbconnect(username, password)

    def connect2(username, password):
        db2.dbconnect(username, password)

虽然两个脚本都已经被加密，但是用户可以自己写一个简单的脚本来代替加密的 `foo.py`

.. code:: python

    def connect(username, password):
        print('password is %s', password)

然后调用加密的主脚本 `main.py` ，虽然功能不能正常完成，但是敏感数据却被泄露。

为了避免这种情况发生，需要使用插件为函数 `stare_server` 提供修饰函数，在每次运行
`start_server` 之前，检查被调用的函数是否是自己定义的函数，如果不是，直接抛出异
常。

现在，编辑脚本 `main.py` ，如下所示的方式增加两个内联插件桩

.. code:: python

    import foo

    # PyArmor Plugin: from pytransform import assert_armored

    # PyArmor Plugin: @assert_armored(foo.connect, foo.connect2)
    def start_server():
        foo.connect('root', 'root password')

然后在启用插件模式加密脚本::

    pyarmor obfuscate --plugin on main.py

加密后的脚本相当于下面的代码

.. code:: python

    import foo

    from pytransform import assert_armored

    @assert_armored(foo.connect, foo.connect2)
    def start_server():
        foo.connect('root', 'root password')

这样，在调用 ``start_server`` 之前，修饰函数 ``assert_assert`` 会检查两个
``connect`` 函数，如果它们没有被加密，就会抛出异常。

当然，你也可以自己调用 :func:`check_armored` 来检查并进行处理

.. code:: python

    import foo

    from pytransform import check_armored

    def start_server():

        if not check_armored(foo.connect, foo.connect2):
            print('Found hacker')
            return

        foo.connect('root', 'root password')

.. _call pyarmor from python script:

在 Python 脚本内部调用 `pyarmor`
--------------------------------

在 Python 脚本内部，也可以直接调用 `pyarmor` ，不需要使用 `os.exec` 或者
`subprocess.Popen` 等命令行的方式。例如

.. code-block:: python

    from pyarmor.pyarmor import main as call_pyarmor
    call_pyarmor(['obfuscate', '--recursive', '--output', 'dist', 'foo.py'])

使用选项 ``--silent`` 可以不显示所有输出

.. code-block:: python

    from pyarmor.pyarmor import main as call_pyarmor
    call_pyarmor(['--silent', 'obfuscate', '--recursive', '--output', 'dist', 'foo.py'])

从 v5.7.3 开始，如果以这种方式调用 `pyarmor` 出现了错误，会抛出异常，而不是调用
`sys.exit` 直接退出。

通过 Web API 生成许可文件
~~~~~~~~~~~~~~~~~~~~~~~~~

如果需要使用 Web API 为加密脚本生成许可，可以使用下面的方式生成许可证。它返回一
个字符串，而不是直接写入到文件，这样可以把字符串返回给客户端进行处理。

.. code-block:: python

    from pyarmor.pyarmor import licenses as generate_license_key
    lickey = generate_license_key(name='reg-001',
                                  expired='2020-05-30',
                                  bind_disk='013040BP2N80S13FJNT5',
                                  bind_mac='70:f1:a1:23:f0:94',
                                  bind_ipv4='192.168.121.110',
                                  bind_data='any string')
    print('Generate key: %s' % lickey)

如果需要使用一个 API 为多个产品生成许可，那么需要使用参数 `home` 来区别不同产品
的注册信息。例如

.. code-block:: python

    from pyarmor.pyarmor import licenses as generate_license_key
    lickey = generate_license_key(name='product-001',
                                  expired='2020-06-15',
                                  home='~/.pyarmor-1')
    print('Generate key for product 1: %s' % lickey)

    lickey = generate_license_key(name='product-002',
                                  expired='2020-05-30',
                                  home='~/.pyarmor-2')
    print('Generate key for product 2: %s' % lickey)

.. _check license periodly when the obfuscated script is running:

运行加密脚本的时候周期性的检查许可文件
--------------------------------------

通常情况下只有运行脚本开始启动的时候会检查许可文件，一旦运行之后，就不在检查。从
v5.9.3 之后，实现了在脚本运行过程中对许可文件进行周期性（每小时一次）检查的功能。
所需要做的是使用选项 ``--enable-period-mode`` 生成一个新的许可文件，覆盖默认的许
可文件即可。例如::

    pyarmor obfuscate foo.py
    pyarmor licenses --enable-period-mode code-001
    cp licenses/code-001/license.lic ./dist

需要注意的是只有当调用一个加密函数的时候，才会重新检查许可证。如果在循环里面没有
调用任何函数，那么许可证不会被重新检查。

.. _work with nuitka:

使用 Nuitka 发布加密脚本
------------------------

因为加密后的脚本也是正常的 Python 脚本（外加运行辅助包 ``pyrtransform`` ），所以
完全可以使用 Nuitka 对加密脚本进行处理，就像正常的 Python 脚本一样。但是加密脚本
的时候，需要指定额外的选项 ``--restrict 0`` 和 ``--no-cross-protection`` ，
否则加密脚本可能会报错。例如，首先加密脚本 ``foo.py``::

    pyarmor obfuscate --restrict 0 --no-cross-protection --package-runtime 0 foo.py

然后使用 Nuitka 把加密后的脚本转换成为可执行的文件::

    cd ./dist
    python -m nuitka --include-package pytransform foo.py
    ./foo.bin

需要注意的是一旦脚本被加密之后，Nuitka 无法自动找到该模块导入的所有相关模块（包）。
为了解决这个问题，首先调用 Nuitka 转换没有加密的脚本，生成相应的 ``.pyi`` 文件，
然后把这个文件拷贝到加密脚本所在的目录，这样就可以正常的转换加密后的脚本。例如::

    # 生成 "mymodule.pyi"
    python -m nuitka --module mymodule.py

    pyarmor obfuscate --restrict 0 --no-bootstrap --package-runtime 0 mymodule.py
    cp mymodule.pyi dist/

    cd dist/
    python -m nuitka --module mymodule.py

但是这种方式可能基本没有使用 Nuitka 转换脚本的功能，所以在性能上提升应该不会太明显。

.. note::

   只要 Nuitka 还连接到 CPython 的库来执行转换后的 C 代码，pyarmor 就应该可以和
   Nuitka 共存。但是 Nuitka 的官网上有一段对未来特征的描述::

       It will do this - where possible - without accessing libpython but in C
       with its native data types.

   也就是说，Nuitka 将来不需要 CPython 库，那么在这种情况下， pyarmor 加密的后的
   脚本将无法在 Nuitka 下面执行。


.. _work with cython:

使用 Cython 发布加密脚本
------------------------

下面是一个简单的例子，来说明如何把一个脚本 :file:`foo.py` 使用 `PyArmor`_ 进行加
密，然后在使用 `Cython`_ 把它转换成为扩展模块，脚本的内容如下::

    print('Hello Cython')

首先加密脚本，使用了一些额外的选项，这些选项的作用可参考命令 :ref:`obfuscate` 中
的说明，加密后的脚本和运行辅助文件存放在目录 :file:`dist` 下面::

    pyarmor obfuscate --package-runtime 0 --no-cross-protection --restrict 0 foo.py

接下来使用 `cythonize` 把 `foo.py` 和 `pytransform.py` 转换成为 ``.c`` 文件::

    cd dist
    cythonize -3 -k --lenient foo.py pytransform.py

注意需要使用选项 ``-k`` 和 ``--lenient`` ，否则会报错::

    undeclared name not builtin: __pyarmor__

然后把 `foo.c` and `pytransform.c` 编译成为扩展模块，在 MacOS 平台下，直接执行下
面的命令就可以；如果是 Linux p平台，需要把额外的编译选项 ``-fPIC`` 加入到命令行::

    gcc -shared $(python-config --cflags) $(python-config --ldflags) \
         -o foo$(python-config --extension-suffix) foo.c

    gcc -shared $(python-config --cflags) $(python-config --ldflags) \
        -o pytransform$(python-config --extension-suffix) pytransform.c

最后测试一下这些扩展模块，把所有 `.py` 文件删除了，然后导入加密后的脚本::

    mv foo.py pytransform.py /tmp
    python -c 'import foo'

像预想的那样会在控制台打印出 `Hello Cython`

.. _work with pyupdater:

使用 PyUpdater 发布加密脚本
---------------------------

PyArmor_ 可以使用下面的方式和 PyUpdater_ 一起工作。假设脚本为 `foo.py`:

1. 首先使用 PyUpdater_ 生成 `foo.spec`

2. 其次使用 PyArmor_ 生成 `foo-patched.spec` ，选项 ``--debug`` 可以保留这个中
   间文件在命令执行完成之后不被删除::

    pyarmor pack --debug -s foo.spec foo.py

    # 如果运行可执行文件的时候抛出保护异常，尝试使用下面的命令禁用加密脚本的相关约束
    pyarmor pack --debug -s foo.spec -x " --restrict 0 --no-cross-protection" foo.py

然后 PyUpdater_ 就可以使用这个 `foo-patched.spec` 进行构建。

当脚本修改之后，只需要使用命令 :ref:`obfuscate` 重新加密脚本，加密脚本需要的全部
选项可以在上面 :ref:`pack` 命令执行后的输出中找到。

如果上面生成的包在执行的时候存在问题，可以把 PyArmor 打好的包压缩成 `.zip` 然后
放到 ``/pyu-data/new`` 下面，在这里签名，处理和上传更新包。

更多信息可以查看命令 :ref:`pack` 和 `使用定制的 .spec 文件打包加密脚本`_

.. _binding obfuscated scripts to python interpreter:

.. _绑定加密脚本到固定的 Python 解释器:

绑定加密脚本到固定的 Python 解释器
----------------------------------

为了提高加密脚本的安全性，也可以把加密脚本绑定到某一个固定的 Python 解释器，如果
用户修改了 Python 解释器，那么加密脚本就会直接退出。

当然，在绑定之后，加密脚本的运行环境会有更多约束。有可能目标机器上 Python 的版本
号和加密时候使用的 Python 版本号完全一致才能够运行加密脚本。

如果你使用命令 :ref:`obfuscate` 来加密脚本，那么在加密之后，使用下面的命令生成一
个新的许可文件，覆盖原来的许可文件。例如::

  pyarmor licenses --fixed 1 -O dist/license.lic

这样，加密脚本在运行的时候就会检查当前 Python 的动态库文件，根据平台的不同，它可
能是 pythonXY.dll，libpythonXY.so 或者 libpythonXY.dylib。一旦发现运行环境的动态
库被修改（和加密时候的不一样），加密脚本就会直接退出。

如果你使用的是工程来加密脚本，那么首先生成一个绑定到固定 Python 解释器的许可证文件::

  cd /path/to/project
  pyarmor licenses --fixed 1

默认情况下，它会存放在 `licenses/pyarmor/license.lic` ，然后修改工程配置，使用这
个许可证文件::

  pyarmor config --license=licenses/pyarmor/license.lic

如果是跨平台发布加密脚本，那么还需要额外的工作，要得到目标平台的 Python 动态库的
特征码。首先在目标平台下按照下面的内容创建一个 Python 脚本，然后使用相应的Python
解释器执行这个脚本

.. code:: python

  from ctypes import CFUNCTYPE, cdll, pythonapi, string_at, c_void_p, c_char_p
  from sys import platform


  def get_bind_key():

      if platform.startswith('win'):
          from ctypes import windll
          dlsym = windll.kernel32.GetProcAddressA
      else:
          prototype = CFUNCTYPE(c_void_p, c_void_p, c_char_p)
          dlsym = prototype(('dlsym', cdll.LoadLibrary(None)))

      refunc1 = dlsym(pythonapi._handle, b'PyEval_EvalCode')
      refunc2 = dlsym(pythonapi._handle, b'PyEval_GetFrame')

      size = refunc2 - refunc1
      code = string_at(refunc1, size)

      print('Get bind key: %s' % sum(bytearray(code)))


  if __name__ == '__main__':
      get_bind_key()


运行之后会打印出需要绑定的特征码 `xxxxxx` ，然后使用这个特征码生成相应的新的许可
证::

  pyarmor licenses --fixed xxxxxx -O dist/license.lic

也可以绑定许可文件到多个 Python 解释器，把需要绑定的特征码使用逗号分开即可::

  pyarmor licenses --fixed 1,key2,key3 -O dist/license.lic
  pyarmor licenses --fixed key1,key2,key3 -O dist/license.lic

注意特征码 `1` 可以用来表示当前 Python 解释器。

.. note::

   在32位的 Windows 平台上面无法使用这个特性，因为在不同的机器上甚至每次运行
   Python 的特征码都是不相同的。

.. _customizing cross protection code:

.. _定制交叉保护脚本:

定制交叉保护脚本
----------------

为了保护 PyArmor 核心动态库在运行加密脚本的时候不会被别人修改，默认情况下加密脚
本的时候会在主脚本里面插入保护代码，参考 :ref:`对主脚本的特殊处理` 。但是这种公
开的保护代码总可能会给别人机会来绕过这种保护，所以为了提高安全性，最好的办法是修
改默认的保护代码，使用自己私有的逻辑来保护动态库，这样可以极大的提高安全性。

从 v6.2.0 开始，可以使用命令 :ref:`runtime` 来生成默认的交叉保护代码的脚本，你可
以参考这个脚本来进行修改，使用自己的方法来检查动态库，当然更好的办法是完全写自己
的保护代码，只要能达到在 Python 脚本里面检查动态库 `pytransform` 没有被修改的目
的就可以。

首先使用下面的命令生成默认的交叉保护脚本 ``build/pytransform_protection.py``::

  pyarmor runtime --advanced 2 --output build

然后修改这个生成的脚本，并在加密的时候使用选项 ``--cross-protection`` 来指定这个
脚本就可以了。例如::

  pyarmor obfuscate --cross-protection build/pytransform_protection.py \
                    --advanced 2 foo.py

.. note::

   加密时候的 ``--advanced`` 和生成运行辅助文件时候的 ``--advanced`` 必须一致，
   不同的值生成的运行辅助文件并不相同。

.. _storing runtime file license.lic to any location:

如何把许可文件 license.lic 存放到任意位置
-----------------------------------------

默认情况下运行加密脚本需要的许可文件 `license.lic` 是和模块 :mod:`pytransform`
存放在一起，但是也可以把这个文件放在任意位置，然后在原来的位置创建一个符号链接，指向真正的文件就可以。

在 linux 下面，使用下面的命令把许可文件放到 ``/opt/my_app``::

  ln -s /opt/my_app/license.lic /path/to/obfuscated/pytransform/license.lic

在 Windows 下面，使用下面的命令把许可文件放到 ``C:\Users\Jondy\my_app``::

  mklink \path\to\obfuscated\pytransform\license.lic C:\Users\Jondy\my_app\license.lic

在发布加密包的时候，只要能够设法在安装完成之后，执行下面的脚本就可以放置加密脚本
到任意指定的位置:

.. code:: python

   import os

   def make_link_to_license_file(package_path, target_license="/opt/mypkg/license.lic"):
       license_file = os.path.join(package_path, 'pytransform', 'license.lic')
       if os.path.exists(license_file):
           os.rename(license_file, target_license)
       os.symlink(target_license, license_file)

.. _register multiple pyarmor in same machine:

在同一台机器上注册多个 PyArmor
------------------------------

从 v5.9.0 开始，PyArmor 从环境变量 `PYARMOR_HOME` 读取全局密钥箱和注册信息，默认
的路径是 `~/.pyarmor` 。所以设置环境变量 `PYARMOR_HOME` 指向另外一个路径，然后在
执行命令 `pyarmor` ，就完全可以使用新的注册文件，而不影响原来的命令的注册数据。

最好的方式是使用下面的方式为每一个注册的 PyArmor 创建一个新的命令。

例如，在 Linux 下面，创建一个新的脚本文件 :file:`pyarmor2` ，内容如下

.. code:: bash

    export PYARMOR_HOME=$HOME/.pyarmor_2
    pyarmor "$@"

然后保存在 `/usr/local/pyarmor2` ，并赋予其执行权限::

    chmod +x /usr/local/pyarmor2

在 Windows 下面，则创建一个批处理文件 :file:`pyarmor2.bat`

.. code:: bat

    SET PYARMOR_HOME=%HOME%\another_pyarmor
    pyarmor %*

这样，就可以使用命令 `pyarmor2` 注册新的文件，加密新的工程::

    pyarmor2 register pyarmor-regkey-2.zip
    pyarmor2 obfuscate foo2.py

使用外部脚本获取加密包的许可证信息
----------------------------------

如果有一个加密后的包，如何获取它的发布者以及相关的许可信息呢？只需要在运行辅助包
:mod:`pytransform` 所在的目录执行下面的脚本就可以

.. code-block:: python

   from pytransform import pyarmor_init, get_license_info
   pyarmor_init(is_runtime=1)
   licinfo = get_license_info()
   print('This obfuscated package is issued by %s' % licinfo['ISSUER'])
   print('License information:')
   print(licinfo)

如果是使用超级模式加密的脚本，只要一个扩展模块 `pytransform` ，使用下面的脚本，
和上面的方式基本类似但是更简单

.. code-block:: python

   from pytransform import get_license_info
   licinfo = get_license_info()
   print('This obfuscated package is issued by %s' % licinfo['ISSUER'])
   print('License information:')
   print(licinfo)

在 v6.2.7 之后，还可以直接调用辅助脚本::

  cd /path/to/obfuscated_package
  python -m pyarmor.helper.get_license_info

.. _how to protect data files:

.. _如何保护数据文件:

如何保护数据文件
----------------

这个功能目前还处于试验阶段。

PyArmor 不会加密数据文件，但是可以把数据文件使用脚本文件包裹起来，然后使用约束模
式 4 加密这个数据脚本文件，这样这个脚本就只能被其他加密脚本调用，通过这种方式来
间接保护数据文件。

在 v6.2.7 之后，可以直接使用辅助脚本来生成相应的数据脚本。例如，下面的命令根据数
据文件 `data.txt` 生成对应的数据模块 `data.py`::

    python -m pyarmor.helper.build_data_module data.txt > data.py

接下来使用约束模式 4 来加密这个数据脚本::

    pyarmor obfuscate --exact --restrict 4 --no-runtime data.py

然后就可以使用下面的方式在其他脚本里面读取数据文件的数据::

    import data

    # 这里装载数据文件的内容到内存变量 text
    # 而在退出 with 语句的时候， 会把 text 从内存中清空
    with data.Safestr() as text:
      ...

在 v6.2.7 之前，可以下载这个帮助脚本 `build_data_module.py <https://github.com/dashingsoft/pyarmor/raw/master/src/helper/build_data_module.py>`_ 然后直接运行::

    python build_data_module.py data.txt > data.py

.. _how to remove docstrings:

如何删除函数和方法的说明文档
----------------------------

在命令行设置环境变量 `PYTHONOPTIMIZE=2` 可以删除函数或者方法中的说明文档，例如::

    # In linux
    PYTHONOPTIMIZE=2 pyarmor obfuscate foo.py

    # In Windows
    set PYTHONOPTIMIZE=2
    pyarmor obfuscate foo.py

.. _using restrict mode with threading and multiprocessing:

.. _在约束模块中使用 threading 和 multiprocessing:

在约束模块中使用 threading 和 multiprocessing
---------------------------------------------

如果一个脚本中使用了 :mod:`threading` 和 :mod:`multiprocessing` ，并且使用
:ref:`约束模式` 3 或者 4 进行加密，那么运行的时候会抛出保护异常。因为这两个系统
模块都没有加密，但是它们会直接调用约束模块里面的函数，这会触发保护异常。

一种解决方案是在加密脚本内部继承系统类 `Thread` ，并使用匿名函数覆盖类方法 `run`
，然后使用自定义的类来创建线程，例如

.. code:: python

    from threading import Thread

    class PrivateThread(Thread):

        def lambda_run(self):
            try:
                if self._target:
                    self._target(*self._args, **self._kwargs)
            finally:
                del self._target, self._args, self._kwargs

        run = lambda self : self.lambda_run()

    def foo():
        print('Hello')

    t = PrivateThread(target=foo)
    t.start()

如果已经定义了自己的类来扩展系统类 `Thread` ，那么只需要将方法 `run`
重命名为 `lambda_run` ，另外新增加一个方法 `run` 。例如

.. code:: python

    from threading import Thread

    class MyThread(Thread):

        # def run(self):
        def lambda_run(self):
            ...

        # 使用 `lambda` 重新定义 `run`
        run = lambda self : self.lambda_run()

另外一种解决方案是定义一个公共的代理模块，这个公共模块使用普通约束，让系统模块来
调用代理模块里面的函数，而不是直接调用私有模块里面的函数。

例如，这个示例脚本 ``foo.py`` 就使用了代理模块 ``pub_foo.py`` 来解决约束冲突

.. code:: python

    import multiprocessing as mp
    import pub_foo

    def hello(q):
        print('module name: %s' % __name__)
        q.put('hello')

    if __name__ == '__main__':
        ctx = mp.get_context('spawn')
        q = ctx.Queue()
        # call "proxy_hello" in public module instead private "hello"
        p = ctx.Process(target=pub_foo.proxy_hello, args=(q,))
        p.start()
        print(q.get())
        p.join()

代理模块 ``pub_foo.py`` 的定义如下

.. code:: python

    import foo

    def proxy_hello(q):
        return foo.hello(q)

现在使用不同的约束模式来分别加密这两个脚本::

    pyarmor obfuscate --restrict 3 foo.py

    # 为了不影响其他加密脚本，选项 --exact 和 --no-runtime 都是必须的
    pyarmor obfuscate --restrict 1 --exact --no-runtime pub_foo.py

还有一种解决方案是使用约束模式 1 加密相关的系统模块，例如 :mod:`threading` ，这
样就不会违反约束模式的要求。对于 :mod:`multiprocessing` 来说，不需要加密里面的所
有模块，仅仅加密那些会直接调用约束模块的脚本就可以了。

.. _使用加密脚本直接替换PyInstaller生成的可执行文件:

使用加密脚本直接替换PyInstaller生成的可执行文件
-----------------------------------------------

从 v6.5.5 开始，PyArmor 提供了一个辅助脚本 ``repack.py`` 可以用来直接替换
PyInstaller 生成的可执行文件里面的脚本，这样就可以直观的把里面的脚本替换成为加密
后的脚本。

* 首先是打包脚本，检查确认打包好的脚本可以正常工作，根据具体的脚本，可能会需要设
  置不同的选项，具体使用方法请参考 `PyInstaller 的文档
  <https://pyinstaller.readthedocs.io>`_ ，这里的例子只是说明这一步需要使用
  PyInstaller 生成可执行文件。如果在这一步有问题，请提交问题报告到 `PyInstaller issues
  <https://github.com/pyinstaller/pyinstaller/issues>`_::

    # 使用单目录模式
    pyinstaller foo.py

    # 检查最终的可执行文件能正常运行
    dist/foo/foo.exe

    # 如果更喜欢单文件模块，使用下面的命令打包
    pyinstaller --onefile foo.py

    # 检查最终的可执行文件能正常运行
    dist/foo.exe

* 然后使用 PyArmor 加密脚本，保存到 ``obfdist`` ，检查确认加密脚本可以正确运行。
  这里的例子是使用 :ref:`obfuscate` 进行加密，请根据需要设置必要的选项，除了下面
  注明的之外，其他选项没有使用约束，只要能把脚本加密就行，当然也可以使用工程的方
  式进行加密::

    # 选项 --package-runtime 必须设置为 0
    pyarmor obfuscate -O obfdist --package-runtime 0 foo.py

    # 如果要启用超级模式加密，使用下面的命令
    pyarmor obfuscate -O obfdist --advanced 2 foo.py

    # 检查加密脚本
    python obfdist/foo.py

* 接着使用这个脚本替换打包好的可执行文件，注意使用的 Python 解释器要和
  PyInstaller 使用的一致::

    # 如果是单目录模式，那么
    python repack.py -p obfdist dist/foo/foo.exe

    # 如果是单文件模式，那么
    # python repack.py -p obfdist dist/foo.exe

* 最后使用输出文件 ``foo-obf.exe`` 覆盖原来的文件::

    # 如果是单目录模式，那么
    cp foo-obf.exe dist/foo/foo.exe

    # 如果是单文件模式，那么
    # cp foo-obf.exe dist/foo.exe

加密脚本的目录结构必须和可执行文件中的对应的脚本路径一致，选项 ``-d`` 可以显示可
执行文件中包含的脚本路径信息，调整脚本脚本的结构，使两者保持一致。例如，下面的命
令会显示详细的路径信息::

    python repack.py -d -p obfdist dist/foo.exe

如果调整了加密脚本的结构，请确保主脚本依旧可以正常执行。

.. note::

    在 v6.5.5 之前，需要下载 ``repack.py``

    https://github.com/dashingsoft/pyarmor/raw/master/src/helper/repack.py

    从 v6.5.5 开始，可以直接使用下面的方式运行::

        python -m pyarmor.helper.repack -p obfdist dist/foo


.. _转换加密脚本为扩展模块:

转换加密脚本为扩展模块
----------------------

在 pyarmor 的包里面有一个帮助脚本文件 ``buildext.py`` 可以用来把加密模块转换成为扩展模块

1. 首先使用选项 ``--no-cross-protection`` 和 ``--restrict 0`` 加密脚本，例如::

    pyarmor obfuscate --no-cross-protection --restrict 0 foo.py

2. 然后使用帮助脚本 ``buildext.py`` 转换 ``dist`` 下面的所有加密脚本，例如::

    python buildext.py dist/

使用 ``-i`` 选项可以在生成扩展模块之后把加密脚本删除，这样 ``dist`` 下面的文件就
可以直接发布，例如::

    python buildext.py -i dist/

默认情况下只有 ``dist`` 下面的加密脚本被转换，如果存在多个子目录，可以依次列出所
有子目录，例如::

    python buildext.py dist/ dist/a/ dist/b/

或者在命令行列出所有 ``.py`` 脚本，例如::

    # 在 Linix 环境
    python buildext.py $(find dist/ -name "*.py")

    # 在 Windows 环境
    FOR /R dist\ %i IN (*.py) DO python buildext.py %i

需要注意的是扩展模块并不会执行 ``if __name__ == "__main__"`` 语句块，如果需要执
行这个语句块，可以使用选项 ``-e`` 把脚本直接转换成为可执行文件，例如::

    python buildext.py -e dist/foo.py
    dist/foo.exe

但是这个可执行文件不能脱离当前 Python 环境独立运行，其本质相当于::

    python dist/foo.py

更多功能和选项，请使用 ``-h`` 查看::

    python buildext.py -h

.. note::

    在 v6.6.0 之前，需要下载 ``buildext.py``

    https://github.com/dashingsoft/pyarmor/raw/master/src/helper/buildext.py

    从 v6.6.0 开始，可以直接使用下面的方式运行::

        python -m pyarmor.helper.buildext ...


.. _distributing obfuscated package with pip:

使用 pip 发布加密包
-------------------

这是一个简单的包目录结构::

    .
    └── mylib
        ├── mylib
        │   ├── __init__.py
        │   └── main.py
        └── setup.py

首先使用选项 ``--enable-suffix 1`` 生成具有唯一名称的运行辅助包::

  cd mylib
  pyarmor runtime -O dist/share --enable-suffix 1

然后使用这个运行辅助文件来加密包::

  pyarmor obfuscate --with-runtime @dist/share mylib/__init__.py

接着编辑 ``setup.py`` ，把所有运行辅助文件作为数据文件增加到里面。例如，这个例子
里面假设运行辅助包的名称为 ``pytransform_vax_xxxxxx``

.. code:: python

   setup(name='mylib',
         ...
         packages=['mylib'],
         package_dir={'mylib': 'dist'},
         data_files=[('pytransform_vax_xxxxxx', 'dist/share/pytransform_vax_xxxxxx/*')]
         ...
         )

最后生成发布包::

  python setup.py sdist

.. note::

   不要加密 ``setup.py``

   对于超级模式，运行辅助文件是不一样的，请根据实现文件修改 ``setup.py``

.. _合并多个 Python 版本的加密脚本:

合并多个 Python 版本的加密脚本
------------------------------

这个功能从 v6.8.0 开始支持。

通常情况下，加密脚本是和 Python 版本绑定到一起的，为了能够使用不同版本的 Python
来运行加密脚本，一种解决方案就是首先使用多个版本的 Python 对加密进行加密，然后把
加密后的脚本合并成为一个脚本。

PyArmor 提供了一个辅助脚本 ``merge.py`` 可以用来把不同 Python 版本加密的脚本进行
合并，这样同一个加密脚本就可以被多个 Python 版本执行。

下面是基本的使用方法::

  # 使用 python2.7 加密脚本
  python2.7 pyarmor.py obfuscate -O py27 foo.py

  # 在使用另外一个版本的 python3.8 加密脚本
  python3.8 pyarmor.py obfuscate -O py38 foo.py

  # 然后使用辅助脚本进行合并，可以使用任意版本的 Python
  python merge.py py38/ py27/

  # 合并后结果存放默认在 merged_dist
  ls merged_dist/

对于超级模式，也基本相同::

  # 使用 python2.7 加密脚本
  python2.7 pyarmor.py obfuscate --advanced 2 -O py27 foo.py

  # 在使用另外一个版本的 python3.8 加密脚本
  python3.8 pyarmor.py obfuscate --advanced 2 -O py38 foo.py

  # 然后使用辅助脚本进行合并，可以使用任意版本的 Python
  python merge.py py38/ py27/

  # 合并后结果存放默认在 merged_dist
  ls merged_dist/

.. note::

   如果合并后的加密脚本运行时候出现保护异常，尝试使用加密选项
   ``--no-cross-protection`` 来对脚本进行加密。

.. note::

    在 v6.8.0 之前，需要下载 ``merge.py``

    https://github.com/dashingsoft/pyarmor/raw/master/src/helper/merge.py

    从 v6.8.0 开始，可以直接使用下面的方式运行::

        python -m pyarmor.helper.merge ...

.. include:: _common_definitions.txt
