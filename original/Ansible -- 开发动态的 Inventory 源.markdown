# Ansible -- 开发动态的 Inventory 源

标签（空格分隔）： Ansible Inventory Python 自动化配置管理

---

**目录**

 - 脚本约定
 - 优化外部的 Inventory 脚本

如在 [Dynamic Inventory][1] 描述的，ansible 可以从动态源拉取清单信息（注：主机相关的信息清单），包括云资源。

我们怎样编写一个新的？

简单！我们仅仅需要创建一个脚本或是程序当传入一个合适的参数的时候，可以返回正确格式的 JSON 数据。你可以使用任何语言做这个事情。


## 脚本约定

当使用单一参数 ```--list``` 调用外部节点脚本的时候，脚本必须返回一个 JSON 散列/字典的所有被管理的组。每组的值应该是包含每台主机/IP 列表的散列/字典，可能的子组，和可能的组变量，或者就是一个简单的主机/IP 地址列表，像这样：

```
{
    "databases"   : {
        "hosts"   : [ "host1.example.com", "host2.example.com" ],
        "vars"    : {
            "a"   : true
        }
    },
    "webservers"  : [ "host2.example.com", "host3.example.com" ],
    "atlanta"     : {
        "hosts"   : [ "host1.example.com", "host4.example.com", "host5.example.com" ],
        "vars"    : {
            "b"   : false
        },
        "children": [ "marietta", "5points" ]
    },
    "marietta"    : [ "host6.example.com" ],
    "5points"     : [ "host7.example.com" ]
}
```

在新的 1.0 版本中。

在 1.0 版本之前，每个组仅仅只有主机名/IP 地址列表，像以上的 webservers, marietta, 和 5points 组。

当我们调用 ```--host <hostname>``` 参数的时候（where ```<hostname>``` is a host from above），该脚本必须返回一个空的 JSON 散列/字典，或者是一个用于模板和 ```playbooks``` 的 散列/字典变量。返回变量是可选的，如果脚本不期望这样做，需要返回的就是一个空的散列/字典。

```
{
    "favcolor"   : "red",
    "ntpserver"  : "wolf.example.com",
    "monitoring" : "pack.example.com"
}
```

## 优化外部清单脚本

在新的 1.3 版本中。

上述的库存清单脚本系统细节适用于 Ansible 的所有版本。但是为每台主机调用 ```--host``` 是非常昂贵的，特别是如果它涉及到昂贵的远程子系统调用 API。在 Ansible 的 1.3 或者是更新的版本中，如果清单脚本返回一个叫做 ```_meta```的顶级元素，它可能在一个清单脚本调用中返回所有的主机变量。当这个 meta 元素包含一个值 "hostvars"，库存清单脚本不会为每个主机调用 ```--host```，这将导致有大量主机的时候，性能会有显著的提升，并且使得库存清单脚本生成的客户端缓存更加容易生效。

被添加进顶级 JSON 字典的数据看起来像这样：

```
{

    # results of inventory script as above go here
    # ...

    "_meta" : {
       "hostvars" : {
          "moocow.example.com"     : { "asdf" : 1234 },
          "llama.example.com"      : { "asdf" : 5678 },
       }
    }

}
```

延伸阅读：

- [Python API][2]
- [Developing Modules][3]
- [Developing Plugins][4]
- [Ansible Tower][5]
- [Development Mailing List][6]
- [irc.freenode.net][7]


  [1]: http://docs.ansible.com/intro_dynamic_inventory.html
  [2]: http://docs.ansible.com/developing_api.html
  [3]: http://docs.ansible.com/developing_modules.html
  [4]: http://docs.ansible.com/developing_plugins.html
  [5]: http://ansible.com/ansible-tower
  [6]: http://groups.google.com/group/ansible-devel
  [7]: http://irc.freenode.net/