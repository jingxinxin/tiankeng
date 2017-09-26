t Hooks 进行自动部署
欧雷 发表于 2 年之前0 条评论标签：Git部署网页服务器Linux
昨天开始接手开发公司前端团队的主页，在稍微修改点东西后推送到远程仓库想看下线上结果时发现并没有更改！询问一把手得知，居然还需要连接到服务器执行一下 git pull 才行……对于我这种怕麻烦的人来说，简直不能忍！

经过一番查找资料以及一顿折腾，终于让它能够自动跑起来了，真是高兴得我手舞足蹈啊！虽然弄了较长时间，在实践的过程中踩了点坑，但回过头来一看还是挺简单的。总的来说，就只是在服务器和本机都做一下配置。（这不废话么……）

由于公司的服务器是 CentOS，我所使用的电脑是 Mac OS X，故本文内容是基于这两个系统环境所写。GUI 在给用户带来很多便利的同时也隐藏了一些不便，如：需要下载应用软件及在操作界面交互。鉴于本文的中心是「自动化」，所以一切操作都采用命令行——

远程连接服务器
在搭建环境的整个过程中，有很多步骤是需要连接到服务器进行的，然而在每次访问的时候都需要输入用户名和密码，就像逢年过节回家聚会都会被亲戚朋友询问「什么时候结婚呀」「何时抱小孩啊」。这就是为什么要把这步放到前面——在自己脑门上写上计划的结婚生子时间，省得他们总问！

生成 SSH 密钥

密钥是免登录连接服务器的通行证，有种刷脸通行的感觉。如果本地已经存在并且不想另外生成的话，可以跳过此步。

cd ~/.ssh 切换目录后用 ssh-keygen -t rsa -C "用于区分密钥的标识" 生成一对具有相同名字的密钥（默认为 id_rsa 和 id_rsa.pub）：用于本地的私钥和用于服务器的公钥（有 .pub 扩展名）。

如果私钥名字不是默认的话，需要手动加入到被「认证」的私钥列表中，否则每次连接服务器都会提示输入服务器的密码。在遇到了一些坑（文后有说明）后，我觉得设置 SSH config 最为靠谱！

编辑 ~/.ssh/config 文件（如果不存在则 touch ~/.ssh/config 创建一下），添加以下内容：

Host HOST_ALIAS                       # 用于 SSH 连接的别名，最好与 HostName 保持一致
  HostName SERVER_DOMAIN              # 服务器的域名或 IP 地址
    Port SERVER_PORT                    # 服务器的端口号，默认为 22，可选
      User SERVER_USER                    # 服务器的用户名
        PreferredAuthentications publickey
          IdentityFile ~/.ssh/PRIVATE_KEY     # 本机上存放的私钥路径
            
          服务器端认证

          先用 pbcopy < ~/.ssh/PRIVATE_KEY.pub 将公钥复制到剪贴板；通过 ssh USER@SERVER 访问服务器，这时会提示输入密码（它也许只有这么一次「询问」的机会）；成功登录后 vim ~/.ssh/authorized_keys，在合适的位置 cmd + V 并保存退出（同时 exit 退出 SSH 连接）。
          
          配置 Git 仓库
          创建服务器端仓库
          
          服务器上需要配置两个仓库，一个用于代码中转的远程仓库，一个用于用户访问的本地仓库。**这里的「远程仓库」并不等同于托管代码的「中央仓库」**，这两个仓库都是为了自动同步代码并部署网站而存在。
          
          在存放远程仓库的目录中（假设是 /home/USER/repos）执行 git init --bare BRIDGE_REPO.git 会创建一个包含 Git 各种配置文件的「裸仓库」。
          
          切换到存放用户所访问文件的目录（假设为 /home/USER/www，如果不存在则在 /home/USER 中执行 mkdir www）：
          
          git init
          git remote add origin ~/repos/BRIDGE_REPO.git
          git fetch
          git checkout master
            
          配置 Git Hook
          
          将目录切换至 /home/USER/repos/BRIDGE_REPO.git/hooks，用 cp post-receive.sample post-receive 复制并重命名文件后用 vim post-receive 修改。其内容大致如下：
          
#!/bin/sh
          
          unset GIT_DIR
          
          NowPath=`pwd`
          DeployPath="../../www"
          
          cd $DeployPath
          git pull origin master
          
          cd $NowPath
          exit 0
          使用 chmod +x post-receive 改变一下权限后，服务器端的配置就基本完成了。
          
          更新本机的仓库源
          
          在原有的（托管代码的）仓库上加入刚才所配置的服务器上的远程仓库的地址为源，以后往那个源推送代码后就会自动部署了。
          
          总结
          在搭建环境时并没有一帆风顺，磕磕绊绊遇到不少问题，虽然很多不值得一提，但有的点还是有记录并分享的价值的！
          
          SSH 私钥「认证」
          
          将生成的私钥进行「认证」有不止一种方式，然而，起初我用的是最挫最不靠谱的 ssh-add ~/.ssh/PRIVATE_KEY——只是在当前 session 有效，一重启就又会被「询问」了！
          
          Git「裸仓库」初始化
          
          在初始化过「裸仓库」进到 hooks 目录后可能会发现并没有生成 post-receive.sample 文件，这时你也许会认为当前使用的 Git 版本无法处理这个 hook。慢着，莫急！先不要重装最新版本的 Git，不妨 touch post-receive 手动创建试下，看能不能正常执行。>
