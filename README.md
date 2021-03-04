# 持续交付与集成说明文档

---------------------

## 目录

- [初始环境准备](#%E5%88%9D%E5%A7%8B%E7%8E%AF%E5%A2%83%E5%87%86%E5%A4%87)
    - [安装devops系列命令](#%E5%AE%89%E8%A3%85devops%E7%B3%BB%E5%88%97%E5%91%BD%E4%BB%A4)
    - [各开发环境定义](#%E5%90%84%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83%E5%AE%9A%E4%B9%89)
    - [部署Jira](#%E9%83%A8%E7%BD%B2jira)
    - [部署Jenkins](#%E9%83%A8%E7%BD%B2jenkins)
    - [配置Jenkins与Github自动集成](#%E9%85%8D%E7%BD%AEjenkins%E4%B8%8Egithub%E8%87%AA%E5%8A%A8%E9%9B%86%E6%88%90)
    - [配置Jenkins与Jira自动集成](#%E9%85%8D%E7%BD%AEjenkins%E4%B8%8Ejira%E8%87%AA%E5%8A%A8%E9%9B%86%E6%88%90)
    - [安装Github命令行程序`gh`（不是`git`）](#%E5%AE%89%E8%A3%85github%E5%91%BD%E4%BB%A4%E8%A1%8C%E7%A8%8B%E5%BA%8Fgh%E4%B8%8D%E6%98%AFgit)
    - [配置代码提交规范化插件](#%E9%85%8D%E7%BD%AE%E4%BB%A3%E7%A0%81%E6%8F%90%E4%BA%A4%E8%A7%84%E8%8C%83%E5%8C%96%E6%8F%92%E4%BB%B6)
    - [项目初始化](#%E9%A1%B9%E7%9B%AE%E5%88%9D%E5%A7%8B%E5%8C%96)
- [持续集成](#%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90)
    - [项目新特性开发](#%E9%A1%B9%E7%9B%AE%E6%96%B0%E7%89%B9%E6%80%A7%E5%BC%80%E5%8F%91)
    - [项目版本发布](#%E9%A1%B9%E7%9B%AE%E7%89%88%E6%9C%AC%E5%8F%91%E5%B8%83)
    - [紧急修复发布](#%E7%B4%A7%E6%80%A5%E4%BF%AE%E5%A4%8D%E5%8F%91%E5%B8%83)

---------------------

## 初始环境准备


### 安装devops系列命令

- 运行本项目目录中的`install`脚本即可。


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

- 管理工作流：
    - `To Do`
    - `In Progress`
    - `In Review`
    - `In Testing`
    - `In Staging`
    - `Done`
    
    示意图如下：
    ![工作流示意图](https://github.com/Raychee/boilerplate-devops/blob/master/ci-workflow.png)

- 配置自动化规则：

    - 创建`feature`/`hotfix`分支时，将`To Do`中关联的卡片移至`In Progress`。
        - When：已创建分支
        - If：比较两个值：`{{branch.name}}`包含正则表达式`^(feature|hotfix)/`
        - `状态`等于`To Do`
        - Then：将事务转换为`In Progress`

    - 创建`feature` pull request时，将`In Progress`中关联的卡片移至`In Review`。
        - When：已创建拉取请求
        - If：比较两个值：`{{pullRequest.sourceBranch.name}}`以`feature/`开始
        - And：比较两个值：`{{pullRequest.destinationBranch.name}}`包含正则表达式`develop$`
        - `状态`等于`In Progress`
        - Then：将事务转换为`In Review`

    - 拒绝`feature` pull request时，将`In Review`中关联的卡片移至`In Progress`。
        - When：拉取请求被拒绝
        - If：比较两个值：`{{pullRequest.sourceBranch.name}}`以`feature/`开始
        - And：比较两个值：`{{pullRequest.destinationBranch.name}}`包含正则表达式`develop$`
        - `状态`等于`In Review`
        - Then：将事务转换为`In Progress`

    - 合并`feature` pull request时，将`In Review`中关联的卡片移至`In Testing`。
        - When：已合并拉取请求
        - If：比较两个值：`{{pullRequest.sourceBranch.name}}`以`feature/`开始
        - And：比较两个值：`{{pullRequest.destinationBranch.name}}`包含正则表达式`develop$`
        - `状态`等于`In Review`
        - Then：将事务转换为`In Testing`
      
    - 创建`release` pull request时，自动创建版本并关联卡片至版本，将`In Testing`中关联的卡片移至`In Staging`。
        - When：已创建拉取请求
        - If：比较两个值：`{{pullRequest.sourceBranch.name}}`以`release/`开始
        - And：比较两个值：`{{pullRequest.destinationBranch.name}}`包含正则表达式`master$`
        - Then：创建版本`{{pullRequest.sourceBranch.name.substringAfter("/")}}`
        - And：编辑事务字段`修复版本`为`下一个未发布的版本`
        - `状态`等于`In Testing`
        - Then：将事务转换为`In Staging`

    - 创建`hotfix` pull request时，自动创建版本并关联卡片至版本，将`In Progress`中关联的卡片移至`In Staging`。
        - When：已创建拉取请求
        - If：比较两个值：`{{pullRequest.sourceBranch.name}}`以`hotfix/`开始
        - And：比较两个值：`{{pullRequest.destinationBranch.name}}`包含正则表达式`master$`
        - Then：创建版本`{{pullRequest.sourceBranch.name.substringAfter("/")}}`
        - And：编辑事务字段`修复版本`为`下一个未发布的版本`
        - `状态`等于`In Testing`
        - Then：将事务转换为`In Staging`

    - 拒绝`release` pull request时，将`In Staging`中关联的卡片移至`In Testing`。
        - When：拉取请求被拒绝
        - If：比较两个值：`{{pullRequest.sourceBranch.name}}`以`release/`开始
        - And：比较两个值：`{{pullRequest.destinationBranch.name}}`包含正则表达式`master$`
        - And：比较两个值：`{{pullRequest.sourceBranch.name.substringAfter("/")}}`等于`{{issue.fixVersions.name}}`
        - `状态`等于`In Staging`
        - Then：将事务转换为`In Testing`

    - 拒绝`hotfix` pull request时，将`In Staging`中关联的卡片移至`In Progress`。
        - When：拉取请求被拒绝
        - If：比较两个值：`{{pullRequest.sourceBranch.name}}`以`hotfix/`开始
        - And：比较两个值：`{{pullRequest.destinationBranch.name}}`包含正则表达式`master$`
        - And：比较两个值：`{{pullRequest.sourceBranch.name.substringAfter("/")}}`等于`{{issue.fixVersions.name}}`
        - `状态`等于`In Staging`
        - Then：将事务转换为`In Progress`

    - 合并`release`/`hotfix` pull request时，将`In Staging`中关联的卡片移至`Done`，并发布版本。
        - When：已合并拉取请求
        - If：比较两个值：`{{pullRequest.sourceBranch.name}}`包含正则表达式`^(release|hotfix)/`
        - And：比较两个值：`{{pullRequest.destinationBranch.name}}`包含正则表达式`master$`
        - And：比较两个值：`{{pullRequest.sourceBranch.name.substringAfter("/")}}`等于`{{issue.fixVersions.name}}`
        - `状态`等于`In Staging`
        - Then：将事务转换为`Done`
        - If：JQL事务`fixVersion = {{issue.fixVersions.name}}`全部匹配JQL`status = Done`
        - Then：发布版本`{{issue.fixVersions.name}}`


### 部署Jenkins

1. 运行命令`deploy jenkins`将Jenkins部署进docker swarm。

    - **注意**：Jenkins需要部署在公网可直接访问的网络中（通过一个公网URL可直接打开Jenkins页面），
      否则无法实现与Github和Jira的自动集成。

2. 等待容器启动，并登录对应url进行初始化配置工作。

3. 安装插件：

    - Github
        - 对接配置说明：https://plugins.jenkins.io/github/
        - 安装后，请参考对接配置说明，或者下文的[配置Jenkins与Github自动集成](#配置jenkins与github自动集成)进行插件合理配置。

    - Jira
        - 对接配置说明：https://plugins.jenkins.io/atlassian-jira-software-cloud/
        - 安装后，根据此对接配置说明，或者下文的[配置Jenkins与Jira自动集成](#配置jenkins与jira自动集成)，在接下来配置Jenkins流水线时，
          在合适的位置添加`jiraSendBuildInfo`和`jiraSendDeploymentInfo`命令。

4. 配置Jenkins流水线：

    - 当feature分支提交pull request时，跑单元测试、测试环境的e2e测试。

    - 当develop分支有新commit时（feature分支合并、release分支合并），构建并部署最新develop版本至测试环境。

    - 当release分支提交pull request时，跑单元测试、预发环境的e2e测试，构建并部署最新发布版本至预发环境。  
      **注意测试过程不要污染线上数据！**

    - 当master分支有新commit时（release分支合并、hotfix分支合并），发布新版本，构建并部署最新发布版本至生产环境。

    - 当hotfix分支提交pull request时，跑单元测试，可选跑预发环境的e2e测试，构建并部署最新发布版本至预发环境。  
      **注意测试过程不要污染线上数据！**

    - 手动执行，将当前预发环境版本部署至生产环境。


### 配置Jenkins与Github自动集成

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


### 配置Jenkins与Jira自动集成

1. 在Jira中创建一个Jenkins专属的App OAuth Credential。

    - 在Jira面板中，依次点击：  
      右上角齿轮标记 > `应用程序` > 左侧`OAuth凭证` > 右侧`创建新凭证`
      
    - 在对话框中，输入以下信息并创建：
        - `应用名称`：`Jenkins`
        - `服务器基本URL`：填写Jenkins服务器对公的URL
        - `标识URL`：应用的logo，可不填
        - `权限`：勾选`部署`和`版本`

    - 记录下创建好的`客户ID`和`密钥`。

2. 在Jenkins中用第1步的信息创建一个`secret text credential`。

    - 在Jenkins主页中，依次点击：  
      `Manage Jenkins` > `Manage Credentials` > `domains (global)` > `Add Credentials`

    - 选择`Kind`为`Secret text`，`Secret`为第1步创建的`密钥`（不是`客户ID`），其它选填。创建Credential。

3. 在Jenkins中配置`Jira Software Cloud Integration`。

    - 在Jenkins主页中，依次点击：  
      `Manage Jenkins` > `Configure System` > 找到`Jira Software Cloud Integration` > `Add Jira Cloud Site` > `Jira Cloud Site`
      
    - 在`Credentials`框中填写：
        - `Site name`：项目的Jira面板地址，类似`<project>.atlassian.net`
        - `Client ID`：用第1步创建的`客户ID`
        - `Secret`：选择第2步创建的secret text credential
      
    - 点击`Test connection`确认连接成功。成功后，点击底部`Save`保存配置。

4. 在任何Jenkins pipeline的代码中，在最后适当的环节调用命令即可将构建和部署信息汇报到Jira面板中。

    - 发送构建信息：`jiraSendBuildInfo site: '<project>.atlassian.net'`  

    - 发送部署信息：`jiraSendDeploymentInfo site: '<project>.atlassian.net' environmentId: 'production' environmentName: 'production' environmentType: 'production'`  


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


## 持续集成


### 项目新特性开发

1. 创建属于自己的`feature`分支。

    - `feature`分支的名称应以`feature/<事务编号>`命名，其中`事务编号`指在JIRA中的事务卡片的唯一编号，例如`STORY-1`。

    - 运行命令`ci-create feature/STORY-1`。该命令会基于`develop`分支创建分支`feature/STORY-1`并推送至远端。

    - 这时在Jira面板中，确认能看到卡片`STORY-1`已与此新分支关联。

2. 在`feature`分支中开发内容，并根据规范提交commit。

    - 通过运行命令`git add`（或开发工具里的等价操作）将修改过的文件添加至暂存区。
        - **注意**：  
          不要使用`git commit`（或开发工具里的等价操作）将`add`和`commit`操作合并起来，导致直接提交了commit。

    - 运行命令`git cz`（或`cz`）**规范化**提交commit。

        - 如果没安装`cz`，请先完成[配置代码提交规范化插件](#配置代码提交规范化插件)步骤。
        - 根据提示步骤一步步填写。
            - `type`：commit的类型，新功能开发选`feat`，bug修复选`fix`，文档更新选`doc`，等等。  
              **注意**：如果一次提交中包含多种类型（即开发了功能又修复了bug），把这些分成**多次提交**。
            - `JIRA issue`：写对应Jira事务卡片的编号，不可填错。
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

    - 在Github页面中确认生成的pull request，  
      其标题应自动指定为`feature`分支的名称，  
      其内容则应是该`feature`分支的所包括的所有commit标题的集合。
      
    - 此时，Jenkins应该可以自动检测到并执行单元测试、测试环境的e2e测试。

4. 项目管理员通过浏览器登录Github，处理pull request。

    - 审核代码中遇到的问题，撰写相关review意见，给出给出`Approve`（通过）/`Request changes`（不通过）结论。
        - 如果Jenkins的测试结果为失败，直接不通过。

    - 若审核不通过，开发者继续开发并重新发起审核。
        - `feature`分支开发者继续在本地进行开发，并通过步骤2提交commit。
        - 运行`ci-sync`命令将新的分支变更同步到远端。（无需再进行步骤3的`ci-publish`。）
        - Github上的pull request可以自动识别分支内容的更新，由项目管理员继续进行审核。

    - 若审核通过，项目管理员合并且删除`feature`分支。
        - 点击页面下方`Merge pull request`按钮（或按钮下拉菜单的第一项`Create a merge commit`）。
        - merge commit的标题自动生成为`Merge pull request #x from ...`，保持不变。
        - merge commit的正文自动生成为`feature/STORY-1`，将其**替换**为pull request的正文内容（
          即由所有feature commit的标题组成的项目列表）。
        - 点击`merge`按钮。
        - merge成功后，点击`Delete branch`将`feature`分支删除。

    - pull request被合并后，Jenkins应该可以构建并部署最新develop分支版本至测试环境。

5. 清理本地分支状态。

    - 运行命令`ci-prune`自动检测到已删除的远端`feature`分支，并自动删除对应的本地分支。

    - 运行命令`ci-sync`将`develop`等分支的最新状态同步至本地。

    - 注意，单独运行`ci-sync`不会删除任何分支，所以如果不运行`ci-prune`，
      那么下次在本地的`feature`分支上直接运行`ci-sync`，会报错显示"分支不存在"。


### 项目版本发布

1. 创建`release`分支。

    - `release`分支的名称应以`release/<版本号>`命名，其中`版本号`指下一个准备发布的版本，如`v0.1.0`。

    - 运行命令`ci-create release`。  
      命令无需指定版本，此时会根据上一个已发布的版本自动推断（增加一个小版本号）。
      例如上一版是`v0.1.0`，那么当前命令会创建分支`release/v0.2.0`。  
      也可以人工指定版本，例如`ci-create release/v0.9.5`。除非特别情况，不建议使用。

    - 确认新的`release`分支创建成功，且已同步至远端。

2. 编辑分支改动，并根据规范提交commit。（可选）

    - 此步骤中提交的commit应当仅限于更改项目代码中跟发布版本有关的配置信息，不可进行任何功能性和bug修复性质的提交。

    - 如若提交commit，步骤应与[项目新特性开发](#项目新特性开发)中的提交commit步骤一致，保持良好规范。

3. 提交`release`分支的pull request，测试并发布至预发环境试运行。

    - 运行命令`ci-publish release/v0.2.0`（或者在`feature/v0.2.0`分支状态下直接运行`ci-publish`）
      将分支`release/v0.2.0`分别与`master`和`develop`合并的请求提交为**2个**pull request到Github。

        - 注意，如果`release`分支没有任何新commit，就意味着它和当前`develop`分支内容完全一致，
          那么此时不会创建由`release`合并到`develop`的pull request（也无法创建），
          最终只会创建1个由`release`合并到`master`的pull request。

    - 在Github页面中确认自动生成的pull request：
        - 标题：这次release所包含的所有commit的事务卡片的编号，例如`STORY-1 STORY-2`。  
        - 正文：这次release所包含的所有commit标题，按行分隔。

    - 此时，Jenkins应会自动检测到，并开始跑单元测试、预发环境的e2e测试，在通过后自动构建并部署当前版本到预发环境。
    
4. 在预发环境中试运行一段时间后，项目管理员通过浏览器登录Github，处理pull request。

    - 审核代码和试运行中遇到的问题，撰写相关review意见，给出`Approve`（通过）/`Request changes`（不通过）结论。
        - 如果Jenkins的测试结果为失败，直接不通过。
        - **注意**：对于两个pull request的情况，要么**同时通过**，要么**同时不通过**。不允许结论不一致。

    - 若审核不通过，删除整个`release`分支，**取消发布**。
        - 在Github上点击`Close Pull Request`关闭pull request。
        - 运行命令`ci-delete release/v0.2.0`分支，将其在本地和远端删除。
        - 若要重新发布新的版本，请回到步骤1重新开始。

    - 若审核通过，项目管理员合并`release`分支。
        - 点击页面下方`Merge pull request`按钮（或按钮下拉菜单的第一项`Create a merge commit`）。
        - merge commit的标题自动生成为`Merge pull request #x from ...`，保持不变。
        - merge commit的正文自动生成为`release/v0.2.0`，将其**替换**为pull request的正文内容（
          即由所有feature commit的标题组成的项目列表）。
        - 点击`merge`按钮。(注意，这里**不要**像`feature`分支那样点击`Delete branch`删除分支。)

    - pull request被合并后，Jenkins应自动检测到，并自动执行第5步。

5. 发布版本（应由Jenkins自动完成）

    - 确认此时`master`分支会产生一个新的merge commit。

    - 运行命令`ci-release`，将release版本`v0.2.0`作为标签打在这个新的`master`commit上，同时删除`release/v0.2.0`分支。

    - 此时，Jenkins应自动构建并部署当前的版本至生产环境。

6. 清理本地分支状态。

    - 运行命令`ci-sync`将`master`等分支的最新状态同步至本地。


### 紧急修复发布

紧急修复的研发流程与[项目版本发布](#项目版本发布)基本一致，不同之处在于：

- 创建、同步、发布分支时，原命令中所有`release`的字样改为`hotfix`。如`ci-create hotfix`、`ci-publish hotfix`等。

- 创建出来的`hotfix`分支的名称形式为`hotfix/<版本号>`，例如`hotfix/v0.1.1`，与创建`release`分支类似。
  而不是`feature`分支那种`feature/<事务编号>`的形式。

- 创建出来的`hotfix`分支基于`master`分支，而不是`develop`。

- 创建`hotfix`分支时，无需指定版本，此时会根据上一个已发布的版本自动推断（增加一个补丁版本号）。
  例如上一版是`v0.1.0`，那么当前命令会创建分支`hotfix/v0.1.1`，而不是像`release`分支一样是`release/v0.2.0`。

- 在`hotfix`分支上编辑并提交commit，是必选，而非`release`分支那样可选。没有任何代码改动内容的紧急修复没有意义。

- 如果在发布`hotfix`的同时正在发布`release`，`release`应该**强制取消**，`hotfix`的版本会优先发布至预发环境和生产环境。
  结束后，才可以重新发起新的`release`版本。
