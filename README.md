比特股(BitShares)区块链即服务Ubuntu模板
============================

比特股是一个工业级别的金融智能合约平台。本模板包含运行比特股区块链项目相关的资源，包括：

* 网络节点: witness_node
* 命令行钱包: cli_wallet
* 延迟网络节点: delayed_node
* 可视化轻钱包网站，包含默认nginx配置文件
* 并可选择安装安装水龙头Faucet服务
* 升级脚本程序

内置版本：2.0.160314，发布时间2016年3月14日，请使用升级脚本升级系统。

使用
--

    # 下载模板代码
    git clone https://github.com/AlexChien/bitshares-template
    cd bitshares-template
    
    # install 参数将在全新的系统中安装环境依赖并从源代码库进行项目编译
    # 可以去泡杯咖啡，需要一点时间安装，根据服务器性能不同大约花费30分钟左右
    sudo ./bitshares.sh install
    
    # update 参数将在已安装了bitshares的环境中升级bitshares相关执行文件
    # 并更新web-ui代码
    # 从镜像新建系统的用户，建议执行一次update动作，获取最新的代码版本
    sudo ./bitshares.sh update
    
install 或者 update 执行完成后，witness_node将自动启动，并可以通过http://服务器IP/ 访问可视化web客户端，客户端将使用本机的witness_node作为web socket数据源。水龙头服务默认使用 DACPLAY 项目组提供的服务。可选自行架设水龙头服务。

安装完成后，witness_node, cli_wallet, delayed_node 保存在 `/usr/bin` 目录下。

关于用户
----

如果你是在一个崭新的系统上运行本脚本，请使用 `./bitshares [install|update]` 运行，脚本在需要使用sudu权限时会做提示，无需sudo的操作则以当前用户身份执行。

如果你是已经获得了一个预装的磁盘镜像，那请使用 `sudo ./bitshares [install|update]` 执行

见证人节点 witness_node
------------------
witness_node是BitShares网络中运行的节点程序，并可提供web socket数据，供web轻钱包使用。默认情况下，blockchain的数据目录在 `$/build/blockchain_data` 下。

相关命令

    # 启动
    sudo /etc/init.d/witness_node start
    # 停止
    sudo /etc/init.d/witness_node stop
    # 检查状态
    sudo /etc/init.d/witness_node status

日志文件位于 `$/build/blockchain_data/logs/witness_node/witness_node.log` 中。查看日志：

    cd bitshares-template
    tail -f build/blochain_data/logs/witness_node/witness_node.log
    

cli_wallet
----------
命令行钱包，用于在服务器上创建使用的钱包，可使用命令行进行交互。

启动钱包

    # wallet_file不存在的话将创建一个
    # -s 连接到本地的位于8090端口的web socket数据源（由witness_node提供）
    # -H cli_wallet本身开放8091端口，供 HTTP-RPC 请求访问
    cli_wallet -w /path_to_wallet_file -s ws://localhost:8090 -H 127.0.0.1:8091
    
设定密码，加密保护钱包文件

    >>> set_password password
    >>> unlock password
    
更多参数及用法请参考 `cli_wallet --help` 或 [CLI Wallet Cookbook](https://github.com/cryptonomex/graphene/wiki/CLI-Wallet-Cookbook)。

TODO: 
增加更多用法示例文档

delayed_node
------------

delayed_node的用法和witness_node几乎一模一样，差别只在于witness_node返回的是实时区块数据，而delayed_node只返回不可逆转的区块数据（大约经过15个区块确认，延时约45秒左右），这样应用代码不需要判断发生软分叉时的重组处理逻辑，可简单放心的使用区块数据。

delayed_node在架构设计中，处于信任节点之后(也就是一个witness_node)之后，获取其数据，处理后通过端口开发供应用程序使用。

![架构图](http://docs.bitshares.org/_images/graphviz-351d386d03191cd0886a1dc08cda77f151a0ff1c.svg)

以下示例中，我们做如下安排：

  * 端口 8090: witness_node (信任节点)
  * 端口 8091: delayed_node
  * 端口 8092: cli_wallet

启动witness_node，作为信任节点

    witness_node --data-dir=trusted_node/ --rpc-endpoint="127.0.0.1:8090"
    
启动delayed_node，返回的交易数据均为为不可逆

    delayed_node --trusted-node="127.0.0.1:8090" \
                 --rpc-endpoint="127.0.0.1:8091"
                 --data-dir delayed_node \

启动cli_wallet，连接到delayed_node

    cli_wallet --server-rpc-endpoint="ws://127.0.0.1:8091" \
               --rpc-http-endpoint="127.0.0.1:8092"

这时，通过HTTP-RPC向cli_wallet请求（端口8092）获取的数据将来自delayed_node，为不可逆交易数据。换句话说，如果当前发生一笔交易到cli_wallet中的某个账户时，此时cli_wallet将看不到这笔交易，等经过大约15个区块，也就是大约45秒之后，才会看到，而这笔交易可视为不可逆转，可进行相关操作。比如用户发起的一笔存款动作。

web-gui
-------

web-gui在build之后，文件放置于$/build/web_root中，nginx配置了默认网站指向该目录。每次build完成后，会将该目录所有者更改为www-data用户，如果你的nginx在其他用户下运行，请做相应修改。

请确保你的云服务器的安全策略或者服务器的防火墙中，打开了80端口，否则web客户端无法访问。系统刚安装完成后，打开http://服务器IP/即可访问，但是可能显示connection error，原因是区块链数据还未完成同步，witness_node尚未能正常提供数据，请等待区块链完成同步。

在/etc/nginx/sites-avaialble目录下，还有https的配置样例，请参考样例配置SSL。

需要注意到是，如果web-gui运行在https下，使用的web socket链接也需要从ws升级到wss。相关服务器配置也请参见示例文件。

水龙头Faucet服务
-----------

比特股系统允许用户使用账户名在网络中进行交互，实际上是在网络上注册账户名，使之与自己掌握的若干公钥进行绑定，这个注册操作需要花费手续费，并且根据名字的长短，花费的手续费也不同。所以在用户第一次使用钱包，注册账户名时，由于账户中还没有BTS币，所以无法完成注册操作。这就需要水龙头服务来替用户支付少量的注册费用，帮助用户立即开始使用系统。

默认使用DACPLAY项目组提供的免费水龙头服务。自建水龙头服务在后续版本中推出。

升级脚本
----

执行以下命令，即可升级bitshares相关节点文件(witness_node, cli_wallet, delayed_node)以及图形化web客户端。

    sudo ./bitshares.sh update
    