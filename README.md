# 持续交付与集成说明文档

---------------------

## 初始环境准备


### 部署Jira

- 配置自动化规则：

    - 创建分支时将`To Do`中关联的卡片移至`In Progress`


### 部署Jenkins

1. 运行命令`./deploy --stack jenkins`将Jenkins部署进docker swarm。

2. 等待容器启动，并登录对应url进行初始化配置工作。

3. 安装插件：

    - Github
    - Jira


### 安装Github命令行程序（`hub`，不是`git`）

1. 根据页面指示安装`hub`。

    - 安装指引页：https://github.com/github/hub#installation
    - 主页：https://hub.github.com/

2. 创建一个personal access token从而使用`hub`来登录到Github。

    - 如果未配置登录信息，`hub`运行时会动态提示输入用户名和密码，但流程有bug，是不能用的。详见说明：
      https://github.com/github/hub/issues/2655

    - 在Github用户`jenkins`个人主页里，依次点击：  
      右上角个人头像 > `Settings` > 左侧`Developer settings` > `Personal access tokens` > 右侧`Generate new token`

    - 勾选所有scope（可根据具体情况去掉一些），点击创建token。

    - 保存好token，切勿丢失，丢失无法找回！

    - 照常使用`hub`命令，在下次它提示输入用户名和密码时，用户名正常填写，密码粘贴为token，即可成功登录。


### 配置代码提交规范化插件

1. 确保系统安装`npm`。

2. 安装`commitizen`插件。
    - 运行命令`npm install -g commitizen`。

3. 安装`commitizen`所需的规范配置。
    - 运行命令`npm install -g @digitalroute/cz-conventional-changelog-for-jira`。

4. 将安装好的规范配置指派给插件。
    - 运行命令`echo '{ "path": "@digitalroute/cz-conventional-changelog-for-jira" }' > ~/.czrc`

5. 插件已安装完成。任何时候需要提交代码（`git commit`）时，使用命令`git cz`或`cz`提交代码即可。


### 项目初始化

1. 了解并安装`git flow`。

    - `git flow`的相关材料：
        - 教程（中文）：http://www.ruanyifeng.com/blog/2015/12/git-workflow.html
        - 教程（英文）：https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow
        - 理念（英文）：https://nvie.com/posts/a-successful-git-branching-model/
        - 安装方法：https://github.com/nvie/gitflow/wiki/Installation

    - 安装完成后，确保`git flow`命令能显示出帮助信息。

2. 初始化一个空项目。

    - 在Github上创建一个新的repo，并通过`git clone`克隆到本地。`cd`至项目目录中，此时它应是一个空目录。

    - 运行命令`ci-init`，初始化生成`develop`分支。


### 配置Jenkins自动集成

1. 在Github中注册一个新用户，例如`jenkins`，并把它加入现有team中，赋予读权限，作为Jenkins与Github沟通的标准用户。

2. 给`jenkins`用户创建一个personal access token。

    - 在Github用户`jenkins`个人主页里，依次点击：  
      右上角个人头像 > `Settings` > 左侧`Developer settings` > `Personal access tokens` > 右侧`Generate new token`

    - 勾选`admin:org_hook`、`repo`、`repo:status`这3个scope，点击创建token。

    - 保存好token，切勿丢失，丢失无法找回！

3. 确认Jenkins的安装配置环境正确。

    - 确认安装了`Github`插件。

    - 确认Jenkins能够通过公网直接访问（具有公网ip/域名，或者有个公网反向代理触达Jenkins）。

4. 在Jenkins中用第2步创建的token创建一个`secret text credential`。

    - 在Jenkins主页中，依次点击：  
      `Manage Jenkins` > `Manage Credentials` > `domains (global)` > `Add Credentials`

    - 选择`Kind`为`Secret text`，`Secret`为刚创建的token，其它选填。创建Credential。

5. 在Jenkins全局配置中配置`GitHub Server Config`。

    - 在Jenkins主页中，依次点击：  
      `Manage Jenkins` > `Configure System` > 找到`Github` > `Add Github Server` > `Github Server`

    - 在`Credentials`框中选择刚创建的secret text credential，点击`Test connection`确认连接成功。

    - 点击底部`Save`保存配置。

6. 在任何Jenkins pipeline中，配置勾选`GitHub hook trigger for GITScm polling`，即可触发在push时的自动构建。


### 项目新特性开发

1. 创建属于自己的`feature`分支。

    - `feature`分支的名称应以`feature/<任务编号>`命名，其中`任务编号`指在JIRA中的任务卡片的唯一编号，例如`STORY-1`。

    - 运行命令`ci-create feature/STORY-1`。该命令会创建分支`feature/STORY-1`并推送至远端。

    - 这时在Jira面板中，确认能看到卡片`STORY-1`已与此新分支关联。

2. 在`feature`分支中开发内容，并根据规范提交commit。

    - 通过运行命令`git add`（或开发工具里的等价操作）将修改过的文件添加至暂存区。
        - **注意**：  
          不要使用`git commit`（或开发工具里的等价操作）将`add`和`commit`操作合并起来，导致直接提交了commit。

    - 运行命令`git cz`（或`cz`）**规范化**提交commit。

        - 如果没安装`cz`，请先完成[配置代码提交规范化插件]()步骤。
        - 根据提示步骤一步步填写。
            - `type`：commit的类型，新功能开发选`feat`，bug修复选`fix`，文档更新选`doc`，等等。  
              **注意**：如果一次提交中包含多种类型（即开发了功能又修复了bug），把这些分成**多次提交**。
            - `JIRA issue`：写对应Jira任务卡片的编号，不可填错。
            - `short description`：commit的标题。
            - `longer description`：commit的正文内容（可选）。
            - `breaking change`：如果这次开发内容导致一些功能或接口无法向前兼容，一定要在这里说明。
        - **注意事项**：
            - 如果一次提交中包含多种类型（即开发了功能又修复了bug），需要把这些分成**多次提交**。
            - 所有commit信息均采用**中文**填写。
    
    - 提交commit后，运行命令`ci-sync`即可将`feature`分支推送到远端保存。

3. `feature`分支开发完成，提交pull request。

    - 运行命令`ci-publish feature/STORY-1`（或者在`feature/STORY-1`分支状态下运行`ci-publish`）
      将分支`feature/STORY-1`与`develop`的合并请求提交为pull request到Github。

    - 当`feature`分支包含多个commit时，pull request的标题会自动指定为`feature`分支的**第一个**commit的message。
      若不合适，可以以如下形式指定pull request标题和内容：  
      `ci-publish -m '标题' -m '内容' feature/STORY-1`  
      如果只指定一个`-m`参数，则只指定标题，内容默认为空。详情可运行`ci-publish -h`查看命令具体用法。
        - **注意**：  
          如果指定标题，请**严格**按照现有其它commit规范撰写。
      
    - 在`ci-publish`中涉及`hub`命令的调用，请先确保[安装Github命令行程序]()步骤已完成。

4. 项目管理员通过浏览器登录Github，审核pull request。

    - 审核代码中遇到的问题，撰写相关review意见，给出Approve / Request changes结论。

    - 若审核不通过，继续开发并重新审核。
        - `feature`分支开发者继续在本地进行开发和提交commit。
        - 运行`ci-sync`命令将新的分支变更同步到远端。
        - Github上的pull request可以自动识别分支内容的更新，由项目管理员继续进行审核。

    - 若审核通过，合并且删除`feature`分支。
        - 点击页面下方`Merge pull request`按钮（或按钮下拉菜单的第一项`Create a merge commit`）。
        - 将merge commit的标题和正文**互换**。Github默认内容不够合理。
        - 点击`merge`按钮。
        - merge成功后，点击`Delete branch`将`feature`分支删除。

5. 清理本地分支状态。

    - 运行命令`ci-prune`自动检测到已删除的远端`feature`分支，并自动删除对应的本地分支。

    - 运行命令`ci-sync`将`develop`等分支的最新状态同步至本地。



### 代码提交模板

