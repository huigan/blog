> solana 链近几年的发展突飞猛进🚀🔥，不管是生态还是技术层面都有超过以太坊的趋势，作为web3从业人员，除以太坊外，多了解其他区块链架构，有助于从多个维度思考区块链在未来的应用落地。

进行solana开发首先就是开发环境的安装，mac环境下面安装比较方便，但是很多人可能都是windows环境，本人也是，所以说一下我的安装经验。

#  一、安装 wsl
首先solana不能直接在windows下面直接开发，需要安装wsl，就是linux虚拟机，个人使用起来感觉还是比较好用的，安装wsl需要win10或11，安装方式：

管理员身份打开windows的命令行输入：

```shell
wsl --install
```

默认安装系统是ubuntu，可以运行以下命令查看：

```shell
wsl -l -v
```

![image.png](https://img.learnblockchain.cn/attachments/2025/04/v5F3Em4U680afd1b4e6ad.png)
启动进入系统：（-d 后面Ubuntu是上面的NAME，我同时安装了2个ubuntu系统，第二个我命名为u2）
```shell
wsl -d Ubuntu
```
启动后进入系统
![image.png](https://img.learnblockchain.cn/attachments/2025/04/2lfuGnvk680afd906a6af.png)


> 默认wsl是安装到C盘的，随着后面各种环境的安装，系统会占用非常大的空间，可以将系统迁移至其他盘，首先需要备份：
>
> ```shell
> wsl --export Ubuntu D:\backup\ubuntu.tar
> ```
>卸载当前的 wsl , 后面还是跟你的系统 NAME。以我的列表为例，如果卸载的是第二个，后面就是u2
> ```shell
> wsl --unregister Ubuntu
> ```
> 然后再把之前的备份导入D盘的目录, newUbuntu是新的NAME，不重复即可。
> ```shell
> wsl --import newUbuntu D:\wsl\newUbuntu D:\backup\ubuntu.tar --version 2
> ```
>这样C盘就不会再被占用大量磁盘空间了。

#   二、wsl中安装环境
## **1、ubuntu系统安装solana环境**
下面的命令可以一键安装solana环境，此处会安装rust、solana cli、anchor cli、nvm 、nodejs、yarn，时间会比较长，
 ```shell
curl --proto '=https' --tlsv1.2 -sSfL https://raw.githubusercontent.com/solana-developers/solana-install/main/install.sh | bash
```
> 如果运行上面的命令，卡在某个地方长时间没反应，可能需要你的wsl系统能链接外网（首先你的windows要先能科学上网，并开启全局代理。）。
> 方法：win开始菜单搜索 wsl ，打开wsl配置->选择网络->右上角网络模式选择 Mirrored。
>
> ![image.png](https://img.learnblockchain.cn/attachments/2025/04/FiK0s3ED680b3f5c8df16.png)
>
> ![image.png](https://img.learnblockchain.cn/attachments/2025/04/xD4lzn2I680b3f7703b92.png)
>
> 配置后需要关闭wsl，再启动。
> ```shell
> wsl --shutdown
> ```

## **2、配置全局变量**
安装完成后记得将solana加到PATH变量，不然下次进入Ubuntu提示不识别solana命令。
xxx替换为您的当前系统用户名。

```shell
echo 'PATH="/home/xxx/.local/share/solana/install/active_release/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```
## **3、查看是否成功**
配置完成后可以运行下面的命令查看是否成功：
```shell
rustc --version
solana --version
anchor --version
```

![image.png](https://img.learnblockchain.cn/attachments/2025/04/m5obMQzH680b44483bba4.png)

至此solana环境安装完成，这时候可能有小伙伴要问，环境都安装在了wsl子系统中，我应该咋创建文件开发啊，接下来我们一起来看看。

# 三、VS Code 中开发solana合约

## **1、安装远程扩展包**
Visual Studio Code支持WSL连接，并直接修改linux/ubuntu中的文件，不过需要安装一个扩展包。
[Remote Development - Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack "        Remote Development - Visual Studio Marketplace    ")

点击上面的链接 或 vs code中搜索 “Remote Development”  ，点击install。
安装成功后，点击vs code左侧 Remote explorer按钮。就会看到wsl中的系统列表，选择你要连接的系统，然后Open Folder中就可以打开任意文件夹了。

![image.png](https://img.learnblockchain.cn/attachments/2025/04/igK5ZqcO680b482adc305.png)

## **2、anchor初始化项目**
运行以下命令，初始化一个名字为 counter的solana项目，名字可以随意。
```shell
anchor init counter
```
vs code中打开counter，可以看到以下目录：

![image.png](https://img.learnblockchain.cn/attachments/2025/04/F1sVeTpt680b4a6a7dbb1.png)

合约代码存放在 /programs/counter/src/lib.rs 文件中，后续主要在此文件进中行开发。

## **3、启动一个本地节点**
切换到本地环境
```shell
solana config set --url localhost
```
启动一个本地节点，单独打开一个终端运行，不要关闭。
```shell
solana-test-validator 
```

> 也可以修改运行的端口
> ```shell
> solana-test-validator   --rpc-port 8899
> ```



![image.png](https://img.learnblockchain.cn/attachments/2025/04/28CNGxSD680b4ff062954.png)

## **4、创建一个钱包**
使 Solana CLI 发送一些交易，我们需要创建一个solana钱包，在根目录中的Anchor.toml文件中可以看到 wallet = "~/.config/solana/id.json" ，此路径为默认密钥对路径，所以我们也直接在此生成。
运行以下命令：

 ```shell
 #生成密钥对
solana-keygen new -o ~/.config/solana/id.json

#查看生成的地址
solana address

#获取测试sol
solana airdrop 50

#查看余额
solana balance
 ```



最后配置 program_id 与 Anchor 密钥同步，默认已同步。
 ```shell
 anchor keys sync
 ```


## **5、编译与测试**
/programs 目录中为合约代码目录。
在vs code中打开一个终端，可以看到vs code默认打开的就是wsl子系统中对应的counter目录。
counter目录下运行编译命令：
```shell
anchor build
```

/tests 为测试目录，anchor默认已经创建了counter.ts测试文件，然后运行下面的命令进行测试。
```shell
anchor test --skip-local-validator
```

![image.png](https://img.learnblockchain.cn/attachments/2025/04/B5irztx2680b93a46d18f.png)

> 当运行测试时不加 --skip-local-validator ，Anchor会自动设置一个本地验证器。因为我们已经有一个正在运行，我们可以用这个参数告诉它跳过这一步。

# 总结
我们初步完成了windows下的solana环境搭建，相比mac也比较稳定，可以让更多的人学习solana的相关开发。
下面我们将一起认识项目中各个目录的作用和写一个简单的solana链上合约，进行下一步之前需要先大概了解rust的相关语法和特性，比如变量、数据类型、函数、引用、结构体、生命周期等。

最后文章中如有问题，欢迎留言指正和讨论。
