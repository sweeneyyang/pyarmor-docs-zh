.. _using project:

.. _使用工程:

使用工程
========

工程是一个包含配置文件的目录，可以用来方便的管理加密脚本。

使用工程管理脚本的有下列优点:

* 可以递增式加密脚本，仅仅加密修改过的脚本，适用于需要加密脚本很多的项目
* 定制选择工程包含的脚本文件，而不是一个目录下全部脚本
* 设置加密模式和定制保护代码
* 更加方便的管理加密脚本

.. _managing obfuscated scripts with project:

使用工程管理加密脚本
--------------------

首先使用命令 :ref:`init` 创建一个工程::

    cd examples/pybench
    pyarmor init --entry=pybench.py

这会在当前目录下面创建一个工程配置文件 :file:`.pyarmor_config` 。 也可
以在其他目录创建一个工程::

    pyarmor init --src=examples/pybench --entry=pybench.py projects/pybench

新创建的工程配置文件存放在 `projects/pybench` 。

使用工程的方式一般是切换当前路径到工程目录，然后运行工程相关命令::

    cd projects/pybench
    pyarmor info

使用命令 :ref:`build` 加密工程中包含的所有脚本::

    pyarmor build

当某些脚本修改之后，再次运行 :ref:`build` ，加密这些修改过的脚本::

    pyarmor build

使用命令 :ref:`config` 来修改工程的配置。

设置工程的 ``--manifest`` 选项，可以完全定制工程中所需要加密的脚本，例
如，把 :file:`dist`, :file:`test` 目录下面的所有 `.py` 排除在工程之外::

    pyarmor config --manifest "include *.py, prune dist, prune test"

也可以指定数据文件，数据文件会被直接拷贝到输出目录，例如::

    pyarmor config --manifest "include *.py, include config.json"
    pyarmor build

详细内容请参考后面的章节 `工程配置文件`_ 中关于属性 `manifest` 的说明。

默认情况下 :ref:`build` 仅仅加密修改过的文件，强制加密所有脚本::

    pyarmor build --force

运行加密后的脚本::

    cd dist
    python pybench.py

.. _obfuscating scripts with different modes:

.. _使用不同加密模式:

使用不同加密模式
----------------

配置不同的加密模式::

    pyarmor config --obf-mod=1 --obf-code=0

使用新的加密模式重新加密脚本::

    pyarmor build -B

.. _obfuscating some special scripts with child project:

使用子工程加密需要特殊处理的脚本
--------------------------------

假定在一个工程中大部分的脚本使用约束模式 3 进行加密，但是很少的几个需要使用约束
模式 2 进行加密，那么这时候使用子工程就很方便。

1. 首先在脚本所在路径创建一个工程::

    cd /path/to/src
    pyarmor init --entry foo.py
    pyarmor config --restrict 3

2. 接着拷贝工程配置文件为子工程 `.pyarmor_config-1`::

    cp .pyarmor_config .pyarmor_config-1

3. 然后配置这个子工程，没有主脚本，包含几个特殊脚本，并且约束模式为 2::

    pyarmor config --entry "" \
                   --manifest "include a.py other/path/sa*.py" \
                   --restrict 2 \
                   .pyarmor_config-1

4. 最后依次加密工程和子工程::

    pyarmor build -B
    pyarmor build --no-runtime -B .pyarmor_config-1

一般情况下我们并不需要在工程中把子工程中的脚本排除在外，因为加密子工程之后，输出
目录相同，原来的加密脚本会被新的加密脚本覆盖。

.. _project configuration file:

.. _工程配置文件:

工程配置文件
------------

每一个工程都有一个 JSON 格式的工程配置文件，它包含的属性如下：

- name

    工程名称

- title

    工程标题

- src

    工程所包含脚本的路径，可以是绝对路径，也可以是相对路径，相对于工程
    所在的目录。

* manifest

    选择和设置工程包含的脚本，其支持的格式和 Python Distutils 中的
    MANIFEST.in 是一样的。默认值为 `src` 下面的所有 `.py` 文件::

        global-include *.py

    多个模式使用逗号分开，例如::

        global-include *.py, exclude __mainfest__.py, prune test

    也可以在其中指定数据文件，数据文件会被直接拷贝到输出目录。

    关于所有支持的模式，参考
    https://docs.python.org/2/distutils/sourcedist.html#commands

* is_package

    可用值: 0, 1, None

    主要会影响到加密脚本的保存路径，如果设置为 1，那么输出路径会额外包
    含包的名称。

* restrict_mode

    加密脚本的约束模式，可用值: 0, 1, 2, 3, 4

    默认值为 1，即加密脚本不能被修改。

    参考 :ref:`约束模式`

   .. note::

      属性 `disable_restrict_mode` 从 v5.5.6 之后不再使用，而是转换为等价的
      `restrict_mode` 。

* entry

    工程的主脚本，可以是多个，以逗号分开::

        main.py, another/main.py, /usr/local/myapp/main.py

    主脚本可以是绝对路径，也可以是相对路径，相对于工程的 `src` 路径。

* output

    输出路径，保存加密后的脚本和运行辅助文件，相对于工程配置文件所在的
    目录。

* capsule

    .. warning:: 在 v5.9.0 中已经删除

    工程使用的密钥箱，默认是 :ref:`全局密钥箱` 。

* obf_code

    是否加密每一个函数（代码块），参考 :ref:`代码加密模式`:

        - 0

        不加密

        - 1 （默认）

        加密每一个函数

        - 2

        使用比模式 1 更复杂的算法来加密每一个函数

* wrap_mode

    是否使用 `try..final` 结构包裹原理的代码块，参考 :ref:`代码包裹模式`:

        - 0

        不包裹

        - 1 （默认）

        包裹每一个代码块

* obf_mod

    是否加密整个模块，参考 :ref:`模块加密模式`:

        - 0

        不加密

        - 1 （默认）

        加密模块

* cross_protection

    是否在主脚本插入交叉保护代码:

        - 0

        不插入

        - 1

        插入默认的保护代码，参考 :ref:`对主脚本的特殊处理`

        - 文件名称

        使用文件指定的自定义模板

* runtime_path

    None 或者任何路径名称

    用来告诉加密脚本去哪里装载动态库 `_pytransform` 。

    默认值为 None ， 是指在 :ref:`运行辅助包` 里面。

    主要用于使用打包工具（例如 py2exe）把加密脚本压缩到一个 `.zip` 文
    件的时候，无法正确定位动态库，这时候把 `runtime_path` 设置为空字符
    串可以解决这个问题。

* plugins

    None 或者列表

    用来扩展加密脚本的认证方式，支持多个插件，例如::

        plugins: ["check_ntp_time", "show_license_info"]

    关于插件的使用实例，请参考 :ref:`使用插件扩展认证方式`

* package_runtime

    保存运行辅助文件的方式:

        - 0

        和加密脚本存放在相同目录，这也是 v5.7.0 之前的存放方式

        - 1 （默认值）

        把运行辅助文件作为包存放在子目录 `pytransform` ，这样目录结构更清晰

* enable_suffix

    .. note:: 在 v5.8.7 中新增

    是否生成带有后缀名称的运行辅助包，当需要导入其他用户加密的模块时候，需要生成
    一个唯一后缀的运行辅助包。可用的值:

        - 0 (默认值)

        运行辅助包（模块）的名称没有后缀，固定为 ``pytransform``

        - 1

        运行辅助包（模块）有后缀，例如， ``pytransform_vax_00001``

* platform

    .. note:: 在 v5.9.0 新增

    仅仅在跨平台加密的时候需要，指定加密脚本的运行平台，多个平台使用逗号分开，平
    台名称必须是标准名称。

* license_file

    .. note:: 在 v5.9.0 新增

    使用这个许可文件来替换默认的许可文件，如果为空则使用默认的许可文件。

    如果非空则文件名称必须是 `license.lic` ，相对路径则是基于当前工程文件的路径。

* bootstrap_code

    .. note:: 在 v5.9.0 新增

    控制如何在加密后的主脚本中生成 :ref:`引导代码`:

        - 0

        不要在主脚本插入引导代码

        - 1 (默认)

        在主脚本中插入引导代码，如果主脚本的名称是 ``__init__.py`` ，那么使用包
        含前置的 ``.`` 的相对导入方式，否则使用绝对导入方式。

        - 2

        在主脚本插入引导代码的时候总是使用绝对导入的方式。

        - 3

        在主脚本插入引导代码的时候总是使用相对导入的方式，会有一个前置的 ``.``

.. include:: _common_definitions.txt
