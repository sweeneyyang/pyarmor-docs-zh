.. _how pyarmor does it:

PyArmor 的工作原理
==================

让我们看看一个普通的 Python 脚本 `foo.py` 加密之后是什么样子。下面是加
密脚本所在的目录 `dist` 下的所有文件列表::

    foo.py

    pytransform/
        __init__.py
        _pytransform.so, or _pytransform.dll in Windows, _pytransform.dylib in MacOS
        pytransform.key
        license.lic

`dist/foo.py` 是加密后的脚本，它的内容如下::

    from pytransform import pyarmor_runtime
    pyarmor_runtime()
    __pyarmor__(__name__, __file__, b'\x06\x0f...')

和加密脚本在一起的目录 `pytransform` 叫做 :ref:`运行辅助包` ，它是运行
加密脚本所必须的。只要这个包能被正常导入进来，加密脚本 `dist/foo.py`
就可以像正常脚本一样被运行。

这是 PyArmor 的一个重要特征： **加密脚本无缝替换原来的脚本**

.. _how to obfuscate scripts:

.. _如何加密脚本:

如何加密脚本
------------

PyArmor 是怎么加密 Python 源代码呢？

首先把源代码编译成代码块::

    char *filename = "foo.py";
    char *source = read_file( filename );
    PyCodeObject *co = Py_CompileString( source, "<frozen foo>", Py_file_input );


接着对这个代码块进行如下处理

* 使用 `try...finally` 语句把代码块的代码段 `co_code` 包裹起来::

    新添加一个头部，对应于 try 语句:

            LOAD_GLOBALS    N (__armor_enter__)     N = length of co_consts
            CALL_FUNCTION   0
            POP_TOP
            SETUP_FINALLY   X (jump to wrap footer) X = size of original byte code

    接着是处理过的原始代码段:

            对于所有的绝对跳转指令，操作数增加头部字节数

            加密修改过的所有指令代码

            ...

    追加一个尾部，对应于 finally 块:

            LOAD_GLOBALS    N + 1 (__armor_exit__)
            CALL_FUNCTION   0
            POP_TOP
            END_FINALLY

* 添加字符串名称 `__armor_enter`, `__armor_exit__` 到 `co_consts`

* 如果 `co_stacksize` 小于 4，那么设置为 4

* 在 `co_flags` 设置自定义的标志位 CO_OBFUSCAED (0x80000000)

* 按照上面的方式递归修改 `co_consts` 中的所有类型为代码块的常量

然后把改装后的代码块转换成为字符串，把字符串进行加密，保护其中的常量和字符串::

    char *string_code = marshal.dumps( co );
    char *obfuscated_code = obfuscate_algorithm( string_code  );

最后生成加密后的脚本，写入到磁盘文件::

    sprintf( buf, "__pyarmor__(__name__, __file__, b'%s')", obfuscated_code );
    save_file( "dist/foo.py", buf );

单纯加密后的脚本就是一个正常的函数调用语句，长得就像这个样子::

    __pyarmor__(__name__, __file__, b'\x01\x0a...')

.. _how to deal with plugins:

.. _如何处理插件:

如何处理插件
------------

在 PyArmor 中插件主要用于在加密的过程中向脚本中注入代码，例如，使用插件在加密脚
本中检查网络时间::

    pyarmor obfuscate --plugin check_ntp_time foo.py

那么为什么不直接把这些代码插入到脚本中呢？主要是因为插件中用到的一些函数必须在加
密脚本中才有效，例如，读取加密脚本中许可文件的有效期等。

每一个插件对应一个 Python 的脚本文件， PyArmor 搜索插件的顺序:

* 如果插件指定了绝对路径，那么直接在这个路径下面查找对应的 `.py` 文件
* 如果是相对路径，那么查找顺序是
    - 当前目录
    - ``$HOME/.pyarmor/plugins``
    - ``{pyarmor_folder}/plugins``
* 没有找到就抛出异常

如果在加密脚本的时候指定了插件，PyArmor 在加密脚本之前，会逐行扫描源代码的注释去
查找插件桩。插件桩分为两种类型：

* 插件定义桩
* 插件调用桩

插件定义桩的格式如下::

    # {PyArmor Plugins}

插件定义桩必须单独占一行，不能有缩进，必须在其他插件调用桩的前面。而且一个脚本中
只能有一个插件定义桩，所有插件对应的脚本文件会被原封不动的插入到下面。需要注意的
是，如果脚本中没有插件定义桩，任何插件定义代码都不会被注入到加密脚本中。

插件调用桩有三种格式，只要注释行的前缀和下面的任意模式匹配，就是一个插件调用桩::

    # PyArmor Plugin:
    # pyarmor_
    # @pyarmor_

插件调用桩可以有缩进，可以在模块的任何地方，但是必须在插件调用桩之后，插件调用桩
可以有任意多个。

第一种格式又称为 `内联调用桩` ，PyArmor 只是简单的把匹配的部分和紧随其后的一个空
格删除，只剩下后半部分的代码。例如，在脚本 ``foo.py`` 有下列内联调用桩::

    # PyArmor Plugin: check_ntp_time()
    # PyArmor Plugin: print('This is plugin code')
    # PyArmor Plugin: if sys.flags.debug:
    # PyArmor Plugin:     check_something():

在加密后的脚本中 ``dist/foo.py`` ，它们将被替换为::

    check_ntp_time()
    print('This is plugin code')
    if sys.flags.debug:
        check_something()

只要在命令行指定了插件，这种替换就会发生。如果没有外部的插件脚本，可以在命令行使
用特殊的插件名称 ``on`` 来生效内联插件桩，例如::

    pyarmor obfuscate --plugin on foo.py

而后面两种格式又称为 `条件调用桩` ，它们只用于调用插件中函数，并且函数名称还必须
和插件名称一致，否则就会被忽略。例如，在使用下面的命令加密脚本::

    pyarmor obfuscate --plugin check_ntp_time foo.py

在 ``foo.py`` 中，第一个条件调用桩会生效，会被替换成为函数调用语句，但是第二个条
件调用桩则不会被替换，还保留原来的注释前缀，因为它调用的函数 ``check_multi_mac``
在命令行中没有对应的插件::

    # pyarmor_check_ntp_time()
    # pyarmor_check_multi_mac()

    ==>

    check_ntp_time()
    # pyarmor_check_multi_mac()

第三种格式和第二种类似，只是把注释前缀替换成为 ``@`` ，主要用于注入修饰函数。例
如::

    # @pyarmor_assert_obfuscated(foo.connect)
    def login(user, name):
        foo.connect(user, name)

    ==>

    @assert_obfuscated(foo.connect)
    def login(user, name):
        foo.connect(user, name)

如果在指定插件名称的时候使用了前缀 ``@`` ，则插件脚本只有在被使用到的情况下，才
会被注入加密脚本中；如果没有被用到，则会被忽略。例如::

    pyarmor obfuscate --plugin @check_ntp_time foo.py

在脚本 ``foo.py`` 中必须使用条件调用桩来调用插件函数::

    # pyarmor_check_ntp_time()

而如果脚本中使用的是下面的内联调用桩，那么插件脚本不会被注入，执行的时候会报错，
找不到函数 ``check_ntp_time``::

    # PyArmor Plugin: check_ntp_time()

.. _special handling of entry script:

.. _对主脚本的特殊处理:

对主脚本的特殊处理
------------------

和其他模块不一样，PyArmor 对主脚本有额外的处理:

* 在加密之前，修改主脚本，插入保护代码
* 在加密之后，修改加密脚本，插入引导代码

在加密主脚本之前，PyArmor 会逐行扫描源代码。如果发现下面的一行::

    # {PyArmor Protection Code}

PyArmor 就会把这一行替换成为保护代码。

如果发现了下面这一行::

    # {No PyArmor Protection Code}

PyArmor 就不会在主脚本中插入保护代码。

如果上面两个特征行都没有，那么在看看有没有这样的行::

    if __name__ == '__main__'

如果有，插入保护代码到这条语句的前面。如果没有，那么不添加保护代码。

默认的保护代码的模板如下::

    def protect_pytransform():

        import pytransform

        def check_obfuscated_script():
            CO_SIZES = 49, 46, 38, 36
            CO_NAMES = set(['pytransform', 'pyarmor_runtime', '__pyarmor__',
                            '__name__', '__file__'])
            co = pytransform.sys._getframe(3).f_code
            if not ((set(co.co_names) <= CO_NAMES)
                    and (len(co.co_code) in CO_SIZES)):
                raise RuntimeError('Unexpected obfuscated script')

        def check_mod_pytransform():
            def _check_co_key(co, v):
                return (len(co.co_names), len(co.co_consts), len(co.co_code)) == v
            for k, (v1, v2, v3) in {keylist}:
                co = getattr(pytransform, k).{code}
                if not _check_co_key(co, v1):
                    raise RuntimeError('unexpected pytransform.py')
                if v2:
                    if not _check_co_key(co.co_consts[1], v2):
                        raise RuntimeError('unexpected pytransform.py')
                if v3:
                    if not _check_co_key(co.{closure}[0].cell_contents.{code}, v3):
                        raise RuntimeError('unexpected pytransform.py')

        def check_lib_pytransform():
            filename = pytransform.os.path.join({rpath}, {filename})
            size = {size}
            n = size >> 2
            with open(filename, 'rb') as f:
                buf = f.read(size)
            fmt = 'I' * n
            checksum = sum(pytransform.struct.unpack(fmt, buf)) & 0xFFFFFFFF
            if not checksum == {checksum}:
                raise RuntimeError("Unexpected %s" % filename)
        try:
            check_obfuscated_script()
            check_mod_pytransform()
            check_lib_pytransform()
        except Exception as e:
            print("Protection Fault: %s" % e)
            pytransform.sys.exit(1)

    protect_pytransform()

在加密脚本的时候， PyArmor 会使用真实的值来替换其中的字符串模板 ``{xxx}``

如果不想让 PyArmor 添加保护代码，除了在脚本中添加上面所示的标志行之外，
也可以使用命令行选项 ``--no-cross-protection`` ，例如::

    pyarmor obfuscate --no-cross-protection foo.py

主脚本被加密之后， PyArmor 会在最前面插入 :ref:`引导代码` 。

.. _how to run obfuscated scripts:

.. _如何运行加密脚本:

如何运行加密脚本
----------------

那么，一个普通的 Python 解释器运行加密脚本 `dist/foo.py` 的过程是什么样呢？

上面我们看到 `dist/foo.py` 的前两行是这个样子::

    from pytransform import pyarmor_runtime
    pyarmor_runtime()

这两行叫做 :ref:`引导代码` ，在运行任何加密脚本之前，它们必须先要被执行。
它们有着重要的使命

* 使用 `ctypes` 来装载动态库 `_pytransform`
* 检查授权文件 `dist/license.lic` 是否合法
* 添加三个内置函数到模块 `builtins`: `__pyarmor__` ， `__armor_enter__` ， `__armor_exit__`

最主要的是添加了三个内置函数，这样 `dist/foo.py` 的下一行代码才不会出错，
因为它马上要调用函数 `__pyarmor__`::

    __pyarmor__(__name__, __file__, b'\x01\x0a...')

`__pyarmor__` 被调用，它的主要功能是导入加密的模块，实现的伪代码如下::

    static PyObject *
    __pyarmor__(char *name, char *pathname, unsigned char *obfuscated_code)
    {
        char *string_code = restore_obfuscated_code( obfuscated_code );
        PyCodeObject *co = marshal.loads( string_code );
        return PyImport_ExecCodeModuleEx( name, co, pathname );
    }

从现在开始，在整个 Python 解释器的生命周期中

* 每一个函数（代码块）一旦被调用，首先就会执行函数 `__armor_enter__` ，
  它负责恢复代码块。其实现原理如下所示::

    static PyObject *
    __armor_enter__(PyObject *self, PyObject *args)
    {
        // Got code object
        PyFrameObject *frame = PyEval_GetFrame();
        PyCodeObject *f_code = frame->f_code;

        // Increase refcalls of this code object
        // Borrow co_names->ob_refcnt as call counter
        // Generally it will not increased  by Python Interpreter
        PyObject *refcalls = f_code->co_names;
        refcalls->ob_refcnt ++;

        // Restore byte code if it's obfuscated
        if (IS_OBFUSCATED(f_code->co_flags)) {
            restore_byte_code(f_code->co_code);
            clear_obfuscated_flag(f_code);
        }

        Py_RETURN_NONE;
    }

* 因为每一个代码块都被人为的使用 `try...finally` 块包裹了一下，所以代码
  块执行完之后，在返回上一级之前，就会调用 `__armor_exit__` 。它会重新加
  密代码块，同时清空堆栈内的局部变量::

    static PyObject *
    __armor_exit__(PyObject *self, PyObject *args)
    {
        // Got code object
        PyFrameObject *frame = PyEval_GetFrame();
        PyCodeObject *f_code = frame->f_code;

        // Decrease refcalls of this code object
        PyObject *refcalls = f_code->co_names;
        refcalls->ob_refcnt --;

        // Obfuscate byte code only if this code object isn't used by any function
        // In multi-threads or recursive call, one code object may be referenced
        // by many functions at the same time
        if (refcalls->ob_refcnt == 1) {
            obfuscate_byte_code(f_code->co_code);
            set_obfuscated_flag(f_code);
        }

        // Clear f_locals in this frame
        clear_frame_locals(frame);

        Py_RETURN_NONE;
    }

.. _how to pack obfuscated scripts:

.. _如何打包加密脚本:

如何打包加密脚本
----------------

虽然加密脚本可以无缝替换原来的脚本，但是打包的时候还是存在一个问题:

**加密之后所有的依赖包无法自动获取**

解决这个问题的基本思路是

1. 使用没有加密的脚本找到所有的依赖文件
2. 使用加密脚本替换原来的脚本
3. 添加加密脚本需要的运行辅助文件到安装包
4. 替换主脚本，因为主脚本会被编译成为可执行文件

PyArmor 提供了一个命令 :ref:`pack` 可以用来直接打包脚本，它会首先加密脚本，然后
调用 PyInstaller 打包，但是在某些情况下，打包可能会失败。这里详细描述了命令
:ref:`pack` 的内部工作原理，可以帮助定位问题所在，同时也可以作为自己直接使用
PyInstaller 打包加密脚本的使用手册。

PyArmor 需要 `PyInstaller` 来完成加密脚本的打包工作，如果没有安装的话，首先执行
下面的命令进行安装::

    pip install pyinstaller

`pyarmor pack` 命令的第一步是加密所有的脚本，保存到 ``dist/obf``::

    pyarmor obfuscate --output dist/obf --package-runtime 0 hello.py

第二步是生成 `.spec` 文件，这是 `PyInstaller` 需要的，把加密脚本需要的运行辅助文
件也添加到里面::

    pyinstaller --add-data dist/obf/license.lic:. \
                --add-data dist/obf/pytransform.key:. \
                --add-data dist/obf/_pytransform.*:. \
                -p dist/obf --hidden-import pytransform
                hello.py

如果是使用超级模式加密脚本::

    pyarmor obfuscate --output dist/obf --advanced 2 --package-runtime 0 hello.py

那么使用下面的命令生成 `.spec` 文件::

    pyi-makespec -p dist/obf --hidden-import pytransform hello.py

.. _note:

    在 Windows 平台下，命令行中的 ``:`` 应该替换为 ``;``

第三步是修改 `hello.spec`, 在 `Analysis` 之后插入下面的语句，主要作用是打包的时
候使用加密后的脚本，而不是原来的脚本::

    src = os.path.abspath('.')
    obf_src = os.path.abspath('dist/obf')

    for i in range(len(a.scripts)):
        if a.scripts[i][1].startswith(src):
            x = a.scripts[i][1].replace(src, obf_src)
            if os.path.exists(x):
                a.scripts[i] = a.scripts[i][0], x, a.scripts[i][2]

    for i in range(len(a.pure)):
        if a.pure[i][1].startswith(src):
            x = a.pure[i][1].replace(src, obf_src)
            if os.path.exists(x):
                if hasattr(a.pure, '_code_cache'):
                    with open(x) as f:
                        a.pure._code_cache[a.pure[i][0]] = compile(f.read(), a.pure[i][1], 'exec')
                a.pure[i] = a.pure[i][0], x, a.pure[i][2]

最后运行这个修改过的文件，生成最终的安装包::

    pyinstaller --clean -y hello.spec

.. note::

   必须要指定选项 ``--clean`` ，否则不会把原来的脚本替换成为加密脚本。

运行打包好的脚本::

   dist/hello/hello.exe

.. include:: _common_definitions.txt
