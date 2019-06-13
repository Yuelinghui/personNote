# ViewModel的使用

`ViewModel`类旨在以生命周期意识的方式存储和管理与UI相关的数据。`ViewModel`类允许数据在`configuration changes`（例如屏幕旋转）后继续存在。

当系统销毁并重建`activity`的时候，可能和UI相关的数据会消失，比如`configuration changes`(例如旋转屏幕)造成的`activity`重建。当然，我们可以在
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTAzNjEzNjI0Myw2NDc1Mzg4NTRdfQ==
-->