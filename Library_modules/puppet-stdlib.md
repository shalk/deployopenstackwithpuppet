# puppet-stdlib

1. [先睹为快](#先睹为快)
2. [核心资源讲解](#核心资源讲解)
3. [小结](##小结)
4. [动手练习 - 光看不练假把式](##动手练习)

**本节作者：余兴超**    
**阅读级别：必读 **  
**阅读时间: 1.5小时**

## 先睹为快
`puppet-stdlib`是由Puppet官方提供的标准库模块。这是一个聚宝盆，几乎在前面介绍的Openstack模块中都会使用到它。因为DSL作为一个不完整的语言（不是男人），缺少某些内置魔法和特性会让程序员们抓狂。  
例如，在Python中借助内置库可以轻松地做数值比较：
```python
max(1,2,3)
```
那么在原生Puppet中，你只能望而兴叹。因此我们需要——puppet-stdlib模块！
```puppet
# 和Python不同的是,max函数须在语句中使用。
$largest=max(1,2,3)
notify {"$largest":}
```

## 核心资源讲解

在这个模块中，它提供了以下Puppet资源：

 * Stages
 * Facts
 * Functions
 * Defined resource types
 * Types
 * Providers

接下来，我们将挑选一些使用频率较高的资源进行讲解。


### Run Stages

我们知道为了保证resources间的执行顺序，可以使用`require`,`subscribe`,`notify`等元参数或者使用链式标记来指定resources间的执行顺序。例如：
```puppet
   package {'ntp':
     ensure => present
   }
   # ntp.conf的配置依赖于ntp软件包的安装
   file {'/etc/ntp.conf':
     ensure  => present,
     require => Package['ntp']
   }
   # ntpd进程的运行依赖于ntp软件包和配置文件
   service {'ntpd':
     ensure    => running,
     subscribe => Package['ntp'],File['/etc/ntp.conf']
   }
```
但是在`class`和`class`之间就没法使用这些方法去标记类之间的执行顺序了。那么`Run stages`允许将指定分组的类按照不同的stage来顺序执行。

#### `main` stage
在Puppet中，默认只有一个stage（`main`）。所有的资源都被默认自动地关联到这个stage上，如果你不显式地为Resources指定stage，那么所有的资源都会在`main stage·阶段允许。

#### 使用定制stage

  使用定制stage和其他资源的调用方式完全相同，除了有一点硬性要求是：
  
> Each additional stage must have an order relationship with another stage

例如，我们可以使用以下方式进行声明：
``` puppet
# 通过元参数的方式
stage { 'first':
  before => Stage['main'],
}
# 通过链式箭头的方式
stage { 'last': }
Stage['main'] -> Stage['last']
```
接下来，我们只需要将stage关联到class：
```puppet
  # stage作为元参数出现
class { 'ntp':
  stage => first,
}
```

#### 使用stdlib::stages
  终于讲到了正题了：`stdlib::stages`类声明了各种run stages用于基础设施，语言运行时和应用的部署。它提供了以下stages：
  
  * setup
  * main
  * runtime
  * setup_infra
  * deploy_infra
  * setup_app
  * deploy_app
  * deploy

使用起来也很简单，不用先声明，直接使用即可：以下为代码示例：
```puppet
  node default {
    include stdlib
    class { java: stage => 'runtime' }
  }
```

### `file_line` type
配置文件的管理是CMS中最主要的目标之一。对于`INI`格式的配置文件的管理方式有多种不同的配置方式。但是对于一些非格式化的配置文件来说，其配置管理通常都是选择使用`template`的方式进行管理。

`file_line` type的出现，使得我们有了一种更轻量的方式去管理非格式化配置文件。它的实现与正则匹配和替换类似。

我们来看看实际的使用吧：
```puppet
# 添加指定行
# 在/etc/sudoers文件中确保`%sudo ALL=(ALL) ALL`被正确添加
file_line { 'sudo_rule':
  path => '/etc/sudoers',
  line => '%sudo ALL=(ALL) ALL',
}
```
在实际使用中，除了添加之外，更多的场景是替换：
```puppet
# 修改指定行
# match参数允许使用正则表达式来精确匹配文本行中的内容
file_line { 'bashrc_proxy':
  ensure => present,
  path   => '/etc/bashrc',
  line   => 'export HTTP_PROXY=http://squid.puppetlabs.vm:3128',
  match  => '^export\ HTTP_PROXY\=',
}
```
在一些场景下，我们需要删除配置文件中指定的行：
```puppet
#删除指定行
#match_for_absence参数决定在ensure => absent下，是否执行操作
#mutiple 确保在多行匹配时继续执行操作（否则报错）
file_line { 'bashrc_proxy':
  ensure            => absent,
  path              => '/etc/bashrc',
  line              => 'export HTTP_PROXY=http://squid.puppetlabs.vm:3128',
  match             => '^export\ HTTP_PROXY\=',
  match_for_absence => true,
  multiple          => true 
}
```
### ensure_packages

`ensure_packages`接受array/hash类型的软件包列表，并确保它们被正确地安装。其实和`package`资源的使用是相似的，但最大的不同点，在于`ensure_packages`函数可以被安全地多次定义，而不会发生duplicated resource的错误。
下面举例说明其使用：
```puppet
# array类型，其中'ksh'被重复传了2次，但可以安全通过编译
ensure_packages(['ksh','openssl'], {'ensure' => 'present'})
ensure_packages(['ksh','vim'], {'ensure' => 'present'})
#
ensure_packages({'ksh' => { enure => '20120801-1' } ,  
                 'mypackage' => 
                          { source => '/tmp/myrpm-1.0.0.x86_64.rpm',                                  provider => "rpm" }}, 
                {'ensure' => 'present'})
```

`ensure_resource`与其类似，这里就不再展开说明。

### validate_xxx

stdlib中提供大量的validate_xxx前缀开头的函数，它们的作用是对输入源进行验证，常被用于变量类型检查中，有点类似于assert的断言，若为false，则退出catalog的编译。

例如: `validate_bool`验证所有传入的参数是否都为true/false布尔类型。
```puppet
$iamtrue = true
validate_bool(true)
validate_bool(true, true, false, $iamtrue)
```
`validate_hash`函数用于验证传入的参数是否为hash类型
```puppet
$my_hash = { 'one' => 'two' }
validate_hash($my_hash)
```
### is_xxx

stdlib中也提供了另一类型的验证函数，仅当验证通过后，会返回true/false的布尔值。 

`is_array`验证传入参数是否为array类型:
```puppet
$numbers=[1,2,3]
is_array($numbers)
```
`is_ipv4_address`验证传入参数是否为ipv4类型的地址:
```puppet
$my_ip='10.0.0.88'
is_ipv4_address($my_ip)
```
### 其他函数

在`puppet-stdlib`模块中还有一些用于处理string，hash，array等类型的函数，我们会在Openstack模块和基础模块中去单独介绍。

## 小结

`puppet-stdlib`模块的出现极大地增强了puppet对于各种数据类型的处理能力，提供诸多功能，请读者详细阅读`puppet-stdlib`的`readme.markdown`文件，或者在线阅读[stdlib的文档](https://forge.puppet.com/puppetlabs/stdlib/readme#usage)。

## 动手练习

1. 验证服务器是否具有eth1，如果有则验证传入的$ipaddress_eth1是否为ipv4类型地址。
2. 将['china','japan','korea']转化为['china-travel','japan-travel','koera-travel']
3. 判断一台服务器是否为虚拟机，如果是则将libvirt_type设置为'qemu',否则设置为'kvm'
4. 从$bigclass namespace中查询变量$var的值，其中$bigclass='foo::bar' (提示:`getvar`)

