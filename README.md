# 持续交付与集成说明文档

---------------------

## 初始环境准备


### 各开发环境定义

- 开发环境`dev`
    - 只部署db相关组件。
    - 用于各个开发者开发时调试用。
    - 环境内的数据可以随意删除、修改。

- 测试环境`test`
    - 完整的一套部署（服务和db），代码版本对应`develop`分支。
    - 用于e2e测试、内部人员试用。**注意避免数据污染！**
        - **数据污染**指自动化测试过程中，由于测试逻辑疏忽或者流程非正常退出，导致测试前后，
          系统中残留测试时的样本数据，或者删除了部分原系统数据，影响系统本身正常运转。
    - 环境内数据**不可以**随意删除、修改，仅能在人工试用中，由应用自身管理数据。
    
- 预发环境`staging`
    - 只部署服务相关组件，不部署db。代码版本对应`master`分支最新待发布版本。
    - 环境内不存储任何数据，所有服务依赖的db直接指向生产环境。
    - 用于e2e测试、人工测试。**注意避免数据污染！**
        - 确认在测试环境中运行的所有自动化测试不存在数据污染后，才可以在此环境运行测试。**切记！**
    
- 生产环境`production`
    - 完整的一套部署（服务和db），代码版本对应`master`分支已发布版本。
  - 用于线上用户直接使用。
  - 环境内数据**不可以**随意删除、修改，仅能由应用自身管理数据。


### 部署Jira

- 配置自动化规则：

    - 创建feature分支时，将`To Do`中关联的卡片移至`In Progress`。

    - 创建pull request时，将`In Progress`中关联的卡片移至`In review`。


### 部署Jenkins

1. 运行命令`./deploy jenkins`将Jenkins部署进docker swarm。

2. 等待容器启动，并登录对应url进行初始化配置工作。

3. 安装插件：

    - Github
        - 对接配置说明：

    - Jira
        - 对接配置说明：https://plugins.jenkins.io/atlassian-jira-software-cloud/

4. 配置Jenkins流水线：

    - 当feature分支提交pull request时，跑单元测试、测试环境的e2e测试。

    - 当develop分支有新commit时（feature分支合并、release分支合并），构建并部署最新develop版本至测试环境。

    - 当release分支提交pull request时，跑单元测试、预发环境的e2e测试（注意测试过程不要污染线上数据）

    - 当master分支有新commit时（release分支合并、hotfix分支合并），构建并部署最新发布版本至预发环境。

    - 当hotfix分支提交pull request时，跑单元测试，可选跑预发环境的e2e测试。

    - 手动执行，将当前预发环境版本部署至生产环境。


### 安装Github命令行程序`gh`（不是`git`）

1. 根据页面指示安装`gh`。

    - 安装指引页：https://github.com/cli/cli#installation
    - 主页：https://cli.github.com/manual/

2. 创建一个personal access token从而使用`gh`来登录到Github。

    - 在Github用户`jenkins`个人主页里，依次点击：  
      右上角个人头像 > `Settings` > 左侧`Developer settings` > `Personal access tokens` > 右侧`Generate new token`

    - 勾选所有scope（可根据具体情况去掉一些），点击创建token。（保存好token，切勿丢失，**丢失无法找回**！）

3. 在终端配置`gh`登录信息。

    - 运行命令`gh auth login`进入登录流程。

    - 选择`log into Github.com`。

    - 登录方式选择`Paste an authentication token`，将刚创建的token粘贴进去确定。

    - 选择一个喜好的协议，比如`HTTPS`。

    - `Authenticate Git with your Github credentials`选择是，即可成功完成登录。


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

1. 了解`git flow`工作流。

    - 教程（中文）：http://www.ruanyifeng.com/blog/2015/12/git-workflow.html
    - 教程（英文）：https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow
    - 理念（英文）：https://nvie.com/posts/a-successful-git-branching-model/

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


## 持续集成


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

    - 提交commit后，可以运行命令`ci-sync`将`feature`分支推送到远端保存。

3. `feature`分支开发完成，提交pull request。

    - 运行命令`ci-publish feature/STORY-1`（或者在`feature/STORY-1`分支状态下直接运行`ci-publish`）
      将分支`feature/STORY-1`与`develop`的合并请求提交为pull request到Github。

    - 确认pull request的标题会自动指定为`feature`分支的名称，内容会将该`feature`分支的commit信息合并。
        
    - 在`ci-publish`中涉及`gh`命令的调用，请先确保[安装Github命令行程序gh]()步骤已完成。

4. 项目管理员通过浏览器登录Github，审核pull request。

    - 审核代码中遇到的问题，撰写相关review意见，给出Approve / Request changes结论。

    - 若审核不通过，开发者继续开发并重新发起审核。
        - `feature`分支开发者继续在本地进行开发，并通过步骤2提交commit。
        - 运行`ci-sync`命令将新的分支变更同步到远端。（无需再进行步骤3的`ci-publish`。）
        - Github上的pull request可以自动识别分支内容的更新，由项目管理员继续进行审核。

    - 若审核通过，项目管理员合并且删除`feature`分支。
        - 点击页面下方`Merge pull request`按钮（或按钮下拉菜单的第一项`Create a merge commit`）。
        - merge commit的标题和内容避免擅自修改。
        - 点击`merge`按钮。
        - merge成功后，点击`Delete branch`将`feature`分支删除。

5. 清理本地分支状态。

    - 运行命令`ci-prune`自动检测到已删除的远端`feature`分支，并自动删除对应的本地分支。

    - 运行命令`ci-sync`将`develop`等分支的最新状态同步至本地。
    
    - 注意，如果不运行`ci-prune`而直接运行`ci-sync`，会报错显示"分支不存在"。



### 代码提交模板

