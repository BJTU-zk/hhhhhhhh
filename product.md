# Coding代码评审模块功能设计
### 参考文档

https://gerrit-documentation.storage.googleapis.com/Documentation/2.15.3/intro-user.html

https://help.github.com/categories/collaborating-with-issues-and-pull-requests



## 代码评审
提供一套权限系统，用户可以用git命令行创建评审，也可以在Web UI上创建评审。评审者在Web UI上评审，评审通过后，由owner将代码合并到目标分支。

> 有权限的用户也可直接将代码push到目标分支，绕开代码评审流程

### 工具
无须额外的工具，常规git客户端即可。创建评审的命令行为：
```bash
git push origin master:refs/for/master
```

> 扩展：
> 1. IDEA/VSCODE插件
> 2. 命令行工具
> 3. electron桌面工具

### 准备工作
客户端需要下载commit-msg文件到工程的.git/hooks目录下。该脚本生成Change-Id补充到commit message里。

### 代码评审流程
每次提交代码后，在合到主分支**之前**做评审。
发起评审的过程，就是`git push origin HEAD:refs/for/<target-branch>`的过程，服务器从而创建一个Change。
服务器端将这个commit放到一个特殊分支（refs/heads/changes/xx/yyyy/zz）里，并在数据库存储元数据（owner、project、target branch等）。

评审者在Web UI上看到新的Change，进行评审，经过某种规则，评审通过（Approved）后，自动或手工合并到目标分支（Submit过程）。

评审过程中，评审者可以直接写comment（或称note），也可以查看文件，在文件中写inline comment。Change的作者可以通过 **amending the commit and uploading the new commit as a new patch set**。

### 术语表
Change: 一个变更，即一个评审，也是一个Conversation。有属性Change-Id、owner、project、target branch等。一个Change包含一到多次PatchSet，其中最新的PatchSet才是有效的，才可能被Submit到目标分支，其它的均为过期的。一个Change还包含多个Comment(或称Note)、Vote。
Change-Id：是一个Change的唯一ID。由客户端的commit-msg hook生成，写在commit message里， 以大写字母I开头。push的commit如果Change-Id在服务器端已存在，说明是创建一个PatchSet到一个已有的Change里，而不用创建新Change。
Change Status：open、ci-xxx-rejected、ci-xxx-approved、WIP、abandoned、closed
机审：机器评审的简称。可通过Web Hook设置CI任务，该CI任务通过status API反馈评审结果。可设置多个CI任务。
人审：评审人手工在Web UI中评审。
审核通过策略：可多种，如一言堂、多人投票、多人中有一人通过/无反对票等。见下面的默认策略模板。
PatchSet：一个Change可以包含一到多个PatchSet，代表最新的代码。（注意：撤回修改后的PatchSet，需要用amend创建，以保证提交消息里包含原来的Change-Id。也可以手工往refs/changes/xxx里 push代码）。
Comment(Note)：对一个Change的评论。可以是直接在UI里的补充Comment，也可以是包含inline notes的评审，包含Vote信息。

### 工程设置

|                                                           | Coding Stage 1 | Coding   | Source       | Github | Gitlab | Gerrit |
| --------------------------------------------------------- | -------------- | -------- | ------------ | ------ | ------ | ------ |
| 工程支持namespace                                         | 有             | 有       | 无           | 有     | 有     | 有     |
| 工程级设置memberships(owner、developer、viewer、reviewer) | 有             | 有       | 有（不全？） | 有     | ?      | ?      |
| 分支级指定membership、策略等（表达式）                    | 有             | 有       | 指定分支名   | 有     | 有     | 有     |
| 分支级设置评审通过策略（见下方表格）                      | 有             | 有       | 无           | 有     | ？     | 有     |
| 分支级设置自动Submit策略（见下方表格）                    | 无             | 有（？） | 无           | 无     | 无     | 无     |
| 分支级设置手动Submit策略（见下方表格）                    | 有             | 有       | 无           | 有     | ？     | 有     |

### 评审通过策略

|                                                              | Coding Stage 1 | Coding | Source       | Github | Gitlab | Gerrit |
| ------------------------------------------------------------ | -------------- | ------ | ------------ | ------ | ------ | ------ |
| 1票通过即通过，1票否决即否决                                 | 有             | 有     | 有           | 有     | 有     | 有     |
| 机审设定                                                     | 无             | 有     | 无           | 有     | 有     | 有     |
| owner强行通过或否决                                          | 有             | 有     | 无           | 有     | ？     | ？     |
| n票通过即通过，1票否决即否决（可设定n值）                    | 无             | 有     | 无           | 有     | ？     | ？     |
| 多数票通过                                                   | 无             | 有     | 无           | 无     | ？     | ？     |
| 评审中更新代码（新PatchSet）                                 | 无             | 有     | 无           | 有     | 有     | 有     |
| 评审基本状态（open、approved、rejected、merged、abandoned、closed） | 有             | 有     | 有（不全？） | 有     | 有     | 有     |

### Submit合并策略

如果既设置了自动Submit策略和手工Submit策略，那么，当approve票数达到标准时，先按自动Submit策略去提交，如果不成功，再提示手工Submit。

手工Submit，需要手工点击”Submit”按钮。

如果按submit policy合并失败，在UI上提示。需要Change Owner按前面步骤在本地rebase，解决冲突，再次发起PatchSet。

|                                                  | Coding Stage 1 | Coding | Source | Github | Gitlab | Gerrit |
| ------------------------------------------------ | -------------- | ------ | ------ | ------ | ------ | ------ |
| Fast Forward Only                                | 有             | 有     | 有     | 有     | 有     | 有     |
| Merge if Necessary                               | 有             | 有     | 无     | 有     | 有     | 有     |
| Rebase if Necessary                              | 有             | 有     | 无     | 有     | 有     | 有     |
| Always Merge                                     | 有             | 有     | 无     | 有     | 有     | 有     |
| Squash                                           | 有             | 有     | 无     | 有     | 有     | 有     |
| Always Merge                                     | 有             | 有     | 无     | 有     | 有     | 有     |
| Always Rebase(Create PatchSet even if can be FF) | 有             | 有     | 无     | 有     | 有     | 有     |
| Cherry Pick                                      | 有             | 有     | 无     | 有     | 有     | 有     |



### 发起评审（即创建Change Conversation）

> 待调研：对于可直接push，无须review的分支，commit-msg是否还生成Change-Id?

Fork/pull:

https://help.github.com/articles/creating-a-pull-request-from-a-fork/



|           | Coding Stage 1 | Coding       | Source | Github       | Gitlab | Gerrit   |
| --------- | -------------- | ------------ | ------ | ------------ | ------ | -------- |
| 来源      | MR/PR/Change   | MR/PR/Change | MR/PR  | PR/MR        | ?      | Change   |
| Change-Id | 无             | 有           | 无     | 用PR的branch | ?      | hook脚本 |

### Dashboard（首页）

首页默认包含 **待评审**、**已评审**、**已关闭**视图，显示全部工程的数据。

> 我们是否需要做这个首页，还是只做工程级的就可以了？

进入某个工程后，也有 **Code Reviews** 的页面，显示该工程下的数据。

> gerrit支持通过表达式定制dashboard，我们不支持！

|                    | Coding Stage 1 | Coding | Source | Github | Gitlab | Gerrit |
| ------------------ | -------------- | ------ | ------ | ------ | ------ | ------ |
| Dashboard基础      | 无             | 有     | 无     | 有     | 有     | 有     |
| Dashboard定制      | 无             | 无     | 无     | 有     | ？     | ？     |
| 工程级CodeReview页 | 有             | 有     | 无     | 有     | ？     | 无？   |

### 评审

评审者在Web UI里查看Code Diff（即一个Change里最新的PatchSet），可以对指定代码行或代码块做comment，也可以做summary comment，提交vote

|                                                              | Coding Stage 1 | Coding | Source       | Github | Gitlab | Gerrit |
| ------------------------------------------------------------ | -------------- | ------ | ------------ | ------ | ------ | ------ |
| 评审中更新代码（新PatchSet）                                 | 无             | 有     | 无           | 有     | 有     | 有     |
| 评审基本状态（open、approved、rejected、merged、abandoned、closed） | 有             | 有     | 有（不全？） | 有     | 有     | 有     |
| 评审扩展状态(wip、private、)                                 | 无             | 有     | 无           | 有     | ?      | 不全   |
| Change之间依赖                                               | 无             | 无     | 无           | 有     | ？     | 有     |
| Change打标签                                                 | 有             | 有     | 无           | 有     | ？     | 有     |
| 关注Change(Conversation)，发送通知                           | 无             | 有     | 无           | 有     | ？     | 有     |
| Comment点赞                                                  | 无             | 有     | 无           | 有     | ？     | 有     |
| Inline Edit                                                  | 无             | 无     | 无           | 无?    | ？     | 有     |

### 评审Comment格式支持

|                                 | Coding Stage 1 | Coding       | Source | Github | Gitlab | Gerrit |
| ------------------------------- | -------------- | ------------ | ------ | ------ | ------ | ------ |
| inline comment                  | 有             | 有           | 有     | 有     | 有     | 有     |
| summary comment                 | 有             | 有           | 无     | 有     | 有     | 有     |
| comment格式支持（markdown）     | 有             | 有           | 无     | 有     | 有     | 有     |
| comment suggestion              | 无             | 无（可以有） | 无     | 有     | ？     | 无     |
| comment格式支持（@、#、CH引用） | 无             | 有           | 无     | 有     | ？     | 部分有 |

markdown引用：
`@mention` 代表一个用户名或team名。如`@liusong64`或`organization/team-name`  

`#reference` 引用一个issue（目前没有，预留）

`CH-123`引用一个ID是123的Change

`:EmojiCode:`代表一个emoji字符，如:+1:

其中，organization-name 是中间用减号隔开部门编码的形式（需要我们指定命名规范）
namespace与organization名称相同

### 修改、重新上传PatchSet

当Change owner看到评审反馈后，修改代码，重新提交、上传新的PatchSet

PR、MR的步骤基本相同，只是`<target-remote>`、`<target-branch>`不一样，都需要amend，以保留Change-Id。

以下是Change的步骤：

```shell
git fetch origin refs/changes/xxxx & git checkout FETCH_HEAD

# 做代码修改...
git add <修改过的文件>
# 用amend来保留commit message，以保持Change-Id
git commit --amend
git push origin HEAD:refs/for/master
```

### 关注及通知(Watch/Unwatch)

评审的owner、reviewers（以及comment里提及的人）**自动关注**该评审。

工程的members可以**手工关注**整个工程或指定评审的变更事件，收到通知。

事件类型有：

* Direct-Push

* Change-Created

* Change-Approved

* Change-Rejected

* Change-New-PatchSet

* Change-Closed

* Change-Submitted

* ...

通知方式有：

* 邮件
* 京ME

> gerrit支持用search express表达关注范围，我们不支持！

> gerrit支持ignore和mute两种，只有微小不同。mute一个change只是mute了最近的一次PatchSet，新的PatchSet还会发通知。我们不需支持？

> 如果Ignore了，就不出现在Incoming Reviews列表里。Review的判定规则也要忽略该评审人。所以，要不我们直接不支持ignore?

### Change级添加评审者

在一个Change的Edit界面，增加评审者（比如某方面专家、利益相关者），并通知他进行评审。这要求评审者细到change级别，并随时间变动。

添加评审者的菜单里，根据git blame得到的用户列表进行智能提示?

### 设定Code Owners规则

.github目录里，建一个CODEOWNERS文件，每行类似

```
*.js   @liusong64
/docs/ @org1/team1
```

当指定的文件（扩展名、目录）修改时，由此推算出code review的owner应该是谁

|                      | Coding Stage 1 | Coding | Source | Github | Gitlab | Gerrit |
| -------------------- | -------------- | ------ | ------ | ------ | ------ | ------ |
| Change级添加评审者   | 无             | 有     | 无     | 有     | ？     | 有     |
| Code Owner的规则设定 | 无             | 有     | 无     | 有     | ？     | 无？   |

### Submit a Change

评审通过后，如果设置了Submit策略为自动，那么系统会自动将Change（中最新的PatchSet）合并到<target-branch>。如果不成功，提示手工Submit，点击"Submit"按钮按手工策略合并，如果成功，自动标为Closed。

如果依然冲突，提示手工解决冲突。


### 手工解决冲突（Rebase a Change）
```
# 拉远程的tracking branch
git fetch
# 拉要改的change的代码并切过去
git fetch origin refs/changes/xxxx & git checkout FETCH_HEAD
git rebase origin/master
# 解决冲突，并标记已解决
git add <resolved-files>
git rebase --continue
# 重新提交PatchSet
git push origin HEAD:refs/for/master
```

注：每次rebase会生成一个新的PatchSet
注：不要rebase已经在<target-branch>上已经存在的commit

### Abandon/Restore a Change
有时评审过程中过现一个Change是坏的，应该放弃掉。这时点击按钮把它标成Abandon状态。
对于Abandon状态的change，可以点击Restore把它恢复成open状态。

#### 用标签分组
标签作为Change的属性，同时可以在Change的页面里显示同一标签下的其它changes，也可作为搜索条件。一般用来标记一个feature或user story。
可以修改change的属性添加、更改topic，也可以在push时指定：

```
git push origin HEAD:refs/for/master -o topic=<the_topic>
```

> gerrit的topic只能有一个，可以包含空格。hashtag可以有多个，类比社交网络里的标签。我们只须支持一类，也许是hashtag？就叫tag ?

#### wip（Work-In-Progres）状态
Wip状态一般等价于open状态，除了不再给reviewers发通知，也不允许reviewers做评审。
一般在下述情况下用：
1. 上传部分代码，并不想被评审
2. 评审中发现问题，标为正在修改

可以在UI上标为ready

#### Private Changes、Ignore、Mutate
> 比如修复安全漏洞的change

### 国际化
中、英文
时间、数字等

---



### 备注

#### Project Owner向导

Project Owner就是一个Project的管理员，负责设置权限、设置评审流程/策略。

> gerrit是把工程的权限配置存储到了git下的一个特殊分支（refs/meta/config）的project.config文件（好处有，可以存修改历史，可以由他人提交MR）。我们需要存DB?


#### 继承(Inheritance) vs 模板(Template)
> gerrit的Project的权限有继承树，根为名All-Projects的工程。设置为inherited值的，从继承树推算。用BLOCK RULE从继承环上去掉某些权限。
> 优点是足够灵活。缺点是树形层次，不直观，运算关联复杂。
>
> 我们考虑用Template，每个namespace可以修改他的Template。创建工程时选择Template，创建成功后，将Template里的设定展开到Project下。

> group owner有其下namespace里所有工程的owner权限
>
> gerrit的group里还可以包含group。可以有多个owner。可以设置可见性（所有人都可见？）

#### Conversation
Conversation template

### 分支表达式（引用）

1. All Branch（直接文字，代表refs/heads/*）
2. Starts With: refs/features/*
3. 指定分支名: refs/features/ha

不准备支持正则表达式 

### Related Changes

用于同时开发多个Feature。

强烈建议开新的分支，基于最新的<target-branch>上开发。
在发起评审前，请先pull <target-branch>再把feature分支rebase上

> gerrit有related changes的概念，用来表达Change的依赖。我们不支持！