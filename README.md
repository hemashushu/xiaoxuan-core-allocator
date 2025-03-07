# XiaoXuan Allocator

一个高效的动态 _heap_ 分配器，主要为 [XiaoXuan Lang](https://www.github.com/hemashushu/xiaoxuan-lang) 而设计。也可用于一般的应用程序或者内核的动态 _heap_ 分配。

特点：

- 为 _XiaoXuan Lang_ 的语言特点而优化；
- 分配和释放内存的速度快；
- 有效降低内部碎片和外部碎片的产生（fragmentation avoidance）。

## License

Copyright (c) 2023 Hemashushu, All rights reserved.

This Source Code Form is subject to the terms of the Mozilla Public License, v. 2.0. If a copy of the MPL was not distributed with this file, You can obtain one at http://mozilla.org/MPL/2.0/.
