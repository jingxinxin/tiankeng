# 基本思路 
客户端开发环境 git push代码到远端服务区的裸仓库，裸仓库执行git hook，让服务端自动找寻到发布的文件地址执行git pull
发布的文件地址git remote指向裸仓库。及你可以视为裸仓库是做为中转站的存在。

## 远程连接服务器

## 生成 SSH 密钥

密钥是免登录连接服务器的通行证，有种刷脸通行的感觉。如果本地已经存在并且不想另外生成的话，可以跳过此步。

cd ~/.ssh 切换目录后用 ssh-keygen -t rsa -C "用于区分密钥的标识" 生成一对具有相同名字的密钥（默认为 id_rsa 和 id_rsa.pub）：用于本地的私钥和用于服务器的公钥（有 .pub 扩展名）。

如果私钥名字不是默认的话，需要手动加入到被「认证」的私钥列表中，否则每次连接服务器都会提示输入服务器的密码。在遇到了一些坑（文后有说明）后，我觉得设置 SSH config 最为靠谱！

编辑 ~/.ssh/config 文件（如果不存在则 touch ~/.ssh/config 创建一下），添加以下内容：

Host HOST_ALIAS                     # 用于 SSH 连接的别名，最好与 HostName 保持一致
HostName SERVER_DOMAIN              # 服务器的域名或 IP 地址
Port SERVER_PORT                    # 服务器的端口号，默认为 22，可选
User SERVER_USER                    # 服务器的用户名
PreferredAuthentications publickey
IdentityFile ~/.ssh/PRIVATE_KEY     # 本机上存放的私钥路径
            
## 服务器端认证

先用 pbcopy < ~/.ssh/PRIVATE_KEY.pub 将公钥复制到剪贴板；通过 ssh USER@SERVER 访问服务器，这时会提示输入密码（它也许只有这么一次「询问」的机会）；成功登录后 vim ~/.ssh/authorized_keys，在合适的位置 cmd + V 并保存退出（同时 exit 退出 SSH 连接）。
          
配置 Git 仓库
创建服务器端仓库
服务器上需要配置两个仓库，一个用于记录代码的裸仓库，一个用于我们自动同步代码的发布仓库。
          
在裸仓库的目录中（假设是 /home/USER/repos）执行 git init --bare BRIDGE_REPO.git 会创建一个包含 Git 各种配置文件的「裸仓库」。
          
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
