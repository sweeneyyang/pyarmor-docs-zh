版本变更日志
============

目前还没有中文版，请参考 `英文版变更日志 </en/latest/change-logs.html>`_

不兼容性说明
------------

1. 加密脚本许可证的兼容性

   **v6.0.1**
   **v7.0.1**

   使用这个版本以及其后的版本生成加密脚本的许可证文件，拷贝到之前老版本加密的脚
   本中，无法正常工作。例如，在最新版本中使用命令 :ref:`licenses` 生成一个新的有
   效期许可证 ``license.lic`` ，如果直接覆盖老版本 v5.9.0 加密脚本中的许可证，那
   么老的加密脚本是无法工作的。解决方案有两个，要么依旧使用老版本生成许可证，要
   么重新加密所有脚本。

   但是，使用旧版本生成的许可证 ``license.lic`` 还是可以继续在新版本中使用。例如，
   使用新版本重新加密所有脚本之后，不需要为老客户重新生成许可证，使用旧版本生成
   的许可证依旧有效。

2. 加密脚本的兼容性

   通常情况下，使用不同版本加密的脚本是可以相互替换的，也就是说，可以使用新版本
   加密的脚本，然后覆盖部分使用老版本的加密脚本，依旧可以工作。

   但是下列不兼容性的版本除外

   **v6.3.0**

   使用这个版本以及其后的版本生成的加密脚本，是无法和老版本的加密脚本混合使用的，
   必须重新加密所有文件并使用新的运行辅助文件，或者还是使用老版本进行加密。
