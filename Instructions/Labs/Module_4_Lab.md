---
lab:
    title: '4：管理 Azure AD 身份验证和授权'
    module: '模块 4：设计身份验证和授权'
---

# 实验室：管理 Azure AD 身份验证和授权
# 学生实验室手册

## 实验场景

作为向 Azure 迁移的一部分，Adatum Corporation 需要定义其标识策略。Adatum 具有名为 adatum.com 的单域 Active Directory 林，并拥有相应的公开注册的 DNS 域。Adatum 企业体系结构团队正在探索将一些本地工作负载转移到 Azure 的选项，其旨在评估 Active Directory 域服务 (AD DS) 环境和目标 Azure 订阅的相关 Azure Active Directory (Azure AD) 租户之间的集成问题，后者是长期身份验证和授权模型的核心组件。

新模型应有利于单一登录，并且可以利用 Azure AD 的多重身份验证功能，按具体应用进行递升式身份验证。为了实现单一登录，体系结构团队计划部署 Azure AD Connect，并采用密码哈希同步配置，从而在两个标识存储区中匹配用户对象。对于希望迁移到云的组织而言，当务之急是选择最佳的身份验证方法。当访问 Azure AD 集成资源时，Azure AD 密码哈希同步是对本地用户实现单一登录身份验证的最简单方法。一些高级 Azure AD 功能（比如标识保护）也需要这种方法。

为了实现递升式身份验证，Adatum 企业体系结构团队打算利用 Azure AD 条件访问策略。条件访问策略支持根据所访问的应用程序或资源类型执行多重身份验证。在完成第一因素身份验证后，将强制执行条件访问策略。条件访问可以基于多个因素，包括：

- 用户或组成员身份。策略可以针对特定的用户和组，从而使管理员可以对访问进行细粒度的控制。
- IP 位置信息。组织可以创建受信任的 IP 地址范围，以便在制定策略决策时使用。管理员可以指定整个国家/地区内流量被阻止或允许的 IP 范围。
- 设备。执行条件访问策略时，可以使用具有特定平台设备或标记有特定状态的用户。
- 应用程序。尝试访问特定应用程序的用户可以触发不同的条件访问策略。
- 实时和经过计算的风险检测。与 Azure AD 标识保护的信号集成允许条件访问策略识别危险的登录行为。然后，策略可以强制用户执行密码更改或多重身份验证，以降低其风险级别，或者阻止用户访问，直到管理员采取手动操作为止。
- Microsoft Cloud App Security (MCAS)。允许实时监视和控制用户应用程序访问和会话，从而提高对云环境内的访问和执行活动的可见性和控制力。

为了实现这些目标，Adatum 企业体系结构团队打算测试其 Active Directory 域服务 (AD DS) 林与 Azure Active Directory (Azure AD) 租户的集成情况，并评估其试点用户的条件访问功能。

## 目标
  
完成本实验室后，你将能够：

 - 部署托管 AD DS 域控制器的 Azure VM

 - 创建和配置 Azure AD 租户

 - 将 AD DS 林与 Azure AD 租户集成


## 实验环境
  
Windows Server 管理员凭据

-  用户名：**Student**

-  密码：**Pa55w.rd1234**

预计用时：120 分钟


## 实验文件

-  \\\\AZ304\\AllFiles\\Labs\\10\\azuredeploy30410suba.json


## 说明

### 练习 0：准备实验室环境

本次练习的主要任务如下：

1. 确定 Azure VM 部署的可用 DNS 名称

1. 使用 Azure 资源管理器快速启动模板，部署运行 AD DS 域控制器的 Azure VM


#### 任务 1：识别 Azure VM 部署的可用 DNS 名称

1. 在实验室计算机上，启动 Web 浏览器，导航至 [Azure 门户](https://portal.azure.com)，然后通过提供要在本实验中使用的订阅中的所有者角色的用户帐户凭据来登录。

1. 在 Azure 门户中，通过直接选择搜索文本框右侧的工具栏图标打开 **“Cloud Shell”** 窗格。

1. 提示选择 **“Bash”** 或 **“PowerShell”** 时，选择 **“PowerShell”**。 

    >**注意**：如果这是你第一次打开 **“Cloud Shell”**，会看到 **“未装载任何存储”** 消息，请选择你在本实验室中使用的订阅，然后选择 **“创建存储”**。 

1. 在“Cloud Shell”窗格中，运行以下命令以识别需要在下一个任务中提供的可用 DNS 名称（将 `<custom-label>` 占位符替换为有可能是全局唯一的任何有效 DNS 主机名，并将 `<Azure region>` 占位符替换为要将托管 Active Directory 域控制器的 Azure VM 部署其中的 Azure 区域的名称）：

    ```powershell
    Test-AzDnsAvailability -DomainNameLabel <custom-label> -Location '<location>'
    ```
      > **注**：要标识可以配置 Azure VM 的 Azure 区域，请参阅 [https://azure.microsoft.com/zh-cn/regions/offers/](https://azure.microsoft.com/zh-cn/regions/offers/) 还可以使用 **Powershell cmdlet** 
      ```powershell
      Get-AzLocation | FT
      ```

1. 验证命令是否返回 **True**。如果没有，请使用不同的 `<custom-label>`  值重新运行相同的命令，直到命令返回为**True** 为止。

1. 记录返回正确结果的 `<custom-label>` 值。需要在下一个任务中使用它。


#### 任务 2：使用 Azure 资源管理器快速启动模板，部署运行 AD DS 域控制器的 Azure VM

1. 在 Azure 门户的“Cloud Shell”窗格工具栏中，选择 **“上传/下载文件”** 图标，在下拉菜单中选择 **“上传”**，然后将文件 **“\\\\AZ304\\AllFiles\Labs\\10\\azuredeploy30410suba.json”** 上传到 Cloud Shell 主目录中。

1. 在“Cloud Shell”窗格中，运行以下命令以创建资源组（将 `<Azure region>` 占位符替换为你在上一个任务中指定的 Azure 区域的名称）：

   ```powershell
   $location = '<Azure region>'
   ```
   ```powershell
   New-AzSubscriptionDeployment `
     -Location $location `
     -Name az30410subaDeployment `
     -TemplateFile $HOME/azuredeploy30410suba.json `
     -rgLocation $location `
     -rgName 'az30410a-labRG'
   ```

1. 在 Azure 门户中，关闭 **“Cloud Shell”** 窗格。

1. 在实验室计算机上，打开另一个浏览器选项卡，然后导航到 [https://github.com/Azure/azure-quickstart-templates/tree/master/active-directory-new-domain](https://github.com/Azure/azure-quickstart-templates/tree/master/active-directory-new-domain)。 

1. 在 **“新建 Windows VM 并新建 AD 林、域和 DC”** 页面上，选择 **“部署到 Azure”**。这将自动将浏览器重定向到 Azure 门户中的 **“使用新的 AD 林创建 Azure VM”** 边栏选项卡。

1. 在 **“使用新的 AD 林创建 Azure VM”** 边栏选项卡中，选择 **“编辑参数”**。

1. 在 **“编辑参数”** 边栏选项卡中，选择 **“加载文件”**，在 **“打开”** 对话框中，选择 **“\\\\AZ304\\AllFiles\Labs\\10\\azuredeploy30410rga.parameters.json”**，然后依次选择 **“打开”** 和 **“保存”**。 

1. 在 **“使用新的 AD 林创建 Azure VM”** 边栏选项卡中，指定以下设置（将其他设置保留位默认值）：

    | 设置 | 数值 | 
    | --- | --- |
    | 订阅 | 在本实验室使用的 Azure 订阅的名称 |
    | 资源组 | **az30410a-labRG** |
    | DNS 前缀 | 你在上一个任务中识别的 DNS 主机名| 

1. 在 **“使用新的 AD 林创建 Azure VM”** 边栏选项卡中，选中复选框 **“我同意上述条款和条件”**，然后选择 **“购买”**。

    > **注意**：不要等待部署完成，而是继续执行下一个练习。该部署可能需要约 15 分钟。在本实验室的第三个练习中，您将使用此任务中部署的虚拟机。


### 练习 1：创建和配置 Azure AD 租户
  
本次练习的主要任务如下：

1. 创建 Azure AD 租户

1. 创建和配置 Azure AD 用户

1. 激活并分配 Azure AD Premium P2 授权


#### 任务 1：创建 Azure AD 租户

1. 在 Azure 门户中，搜索并选择 **“Azure Active Directory”**，然后在 Azure Active Directory 边栏选项卡上选择 **“+ 创建租户”**。

1. 在 **“创建目录”** 边栏选项卡的 **“基本”** 选项卡上，选择 **“Azure Active Directory”** 选项，然后选择 **“下一步: 配置 >”**。

1. 在 **“创建目录”** 边栏选项卡的 **“配置”** 选项卡上，指定以下设置（将其他设置保留其现有值）：

    | 设置 | 数值 |
    | --- | --- |
    | 组织名称 | **Adatum 实验室** |
    | 初始域名 | 由小写字母和数字组成并以字母开头的任何有效 DNS 名称 | 
    | 国家/区域 | **美国** |

   > **注意**： **“初始域名”** 文本框中的绿色复选标记将指示你键入的域名是否有效且唯一。

1. 选择 **“下一步: 查看 + 创建”**，然后选择 **“创建”**。

1. 刷新显示 Azure 门户的浏览器页面，搜索并选择 **“Azure Active Directory”**，然后在 Azure Active Directory 边栏选项卡上选择 **“切换租户”**。

1. 在 **“目录 + 订阅”** 边栏选项卡的 **“Adatum 实验室”** 卡片上选择 “**切换**”。 


#### 任务 2：创建和配置 Azure AD 用户。

1. 在 **“Adatum 实验室 | 概览”** Azure Active Directory 边栏选项卡上的 **“管理”** 部分，选择 **“用户”**，在 **“用户 | 全部用户”** 边栏选项卡上，选择你的用户帐户以显示其 **“配置文件”** 设置。 

1. 在你的用户帐户配置文件边栏选项卡上，选择 **“编辑”**，在 **“设置”** 部分，将 **“使用位置”** 设为 **“美国”** 并保存更改。

    >**注意**：这是必要的，以便稍后在本实验室中将 Azure AD Premium P2 许可分配给你的用户帐户。

1. 导航回 **“用户 - 所有用户”** 边栏选项卡，然后选择 **“+ 新建用户”**。

1. 在 **“新建用户”** 边栏选项卡上，指定以下设置（将其他设置保留为默认值）：

    | 设置 | 数值 |
    | --- | --- |
    | 用户名 | **az30410-aaduser1** |
    | 名称 | **az30410-aaduser1** |
    | 自动生成密码 | 已启用 |
    | 显示密码 | 已启用 |
    | 角色 | **全局管理员** |
    | 使用位置 | **美国** |

    >**注意**：记录完整的用户名（包括域名）和自动生成的密码。您将在此任务中稍后使用它。

1. 在 **“新建用户”** 边栏选项卡上，选择 **“创建”**

1. 在实验室计算机上，打开一个 **“InPrivate”** 浏览器窗口，并使用新创建的 **“az30410-aaduser1”** 用户帐户登录到 [Azure 门户](https://portal.azure.com)。当提示更新密码时，将密码更改为 **“Pa55w.rd1234”**。 

1. 以 **az30410-aaduser1** 用户身份从 Azure 门户注销，并关闭 InPrivate 浏览器窗口。


#### 任务 3： 激活并分配 Azure AD Premium P2 授权

1. 返回到显示 Azure 门户的浏览器窗口，导航至 **“Adatum 实验室”** Azure AD 租户的 **“概述”** 边栏选项卡，在 **“管理”** 部分下，选择 **“许可证”**。

1. 在 **“许可证 | 概述”** 边栏选项卡中，选择 **“所有产品”**，选择 **“+ 试用/购买”**。

1. 在 **“激活”** 边栏选项卡中，在 **“Azure AD Premium P2”** 部分，选择 **“免费试用”**，然后选择 **“激活”**。 

1. 刷新显示 **“许可证 | 所有产品”** 边栏选项卡的浏览器窗口，以验证是否激活成功。 

1. 在 **“许可证 - 所有产品”** 边栏选项卡中，选择 **“Azure Active Directory Premium P2”** 条目。 

1. 在 **“Azure Active Directory Premium P2 | 已授权用户”** 边栏选项卡中，选择 **“+ 分配”**。 

1. 在 **“分配许可证”** 边栏选项卡中，选择 **“用户”**，并在 **“用户”** 边栏选项卡中，选择你的帐户和  **az30410-aaduser1** 用户帐户。

1. 返回到 **“分配许可证”** 边栏选项卡，选择 **“分配选项”**，查看 **“许可证选项”** 边栏选项卡中列出的选项，然后选择 **“确定”**。

1. 在 **“分配许可证”** 边栏选项卡中，选择 **“分配”**。 


### 练习 2：将 AD DS 林与 Azure AD 租户集成
  
本次练习的主要任务如下：

1. 将自定义域名分配到 Azure AD 租户

1. 在 Azure VM 中配置 AD DS

1. 安装 Azure AD Connect

1. 配置已同步用户帐户的属性


#### 任务 1：将自定义域名分配到 Azure AD 租户

1. 在 Azure 门户中，导航到 **“Azure Active Directory  Adatum 实验室” | “概述”** 边栏选项卡。

1. 在 **“Adatum 实验室 | 概述”** 边栏选项卡，选择 **“自定义域名”**。

1. 在 **“Adatum 实验室 | 自定义域名”** 边栏选项卡，识别与 Azure AD 租户关联的主域名和默认 DNS 域名。 

    >**注意**：记录 Azure AD 租户的主 DNS 名称的值。需要在下一个任务中使用它。

1. 在 **“Adatum 实验室 | 自定义域名”** 边栏选项卡，选择 **“+ 添加自定义域”**。

1. 在 **“自定义域名”** 边栏选项卡中，在 **“自定义域名”** 文本框中键入 **“adatum.com”**，然后选择 **“添加域”**。 

1. 在 **“adatum.com”** 边栏选项卡中，查看执行 Azure AD 域名验证所需的信息，然后关闭边栏选项卡。

    > **注意**：因为没有 **adatum.com** DNS 域名，所以将无法完成验证过程。这*不*会阻止你将 **“adatum.com”** Active Directory 域与 Azure AD 租户同步。为此，您将使用 Azure AD 租户的默认主 DNS 名称（其名称以 **onmicrosoft.com** 后缀结尾），该名称在此任务的前期已确定。但是，请记住，因此，Active Directory 域的 DNS 域名和 Azure AD 租户的 DNS 名称将有所不同。这意味着 Adatum 用户在登录 Active Directory 域和登录 Azure AD 租户时需要使用不同的名称。


#### 任务 2： 在 Azure VM 中配置 AD DS

> **注意**：在开始本练习之前，请确保在本实验室开始时启动的 Azure VM 的部署已完成。

1. 在 Azure 门户中，搜索并选择 **“虚拟机”**，并且在 **“虚拟机”** 边栏选项卡，选择 **“az30410a-vm1”**。

1. 在 **“az30410a-vm1”** 边栏选项卡中，选择 **“连接”**，在下拉菜单中选择 **“RDP”**，在 **“RDP”** 选项卡上，其位于 **“az30410a-vm1 | 连接”** 边栏选项卡中，在 **“IP 地址”** 下拉列表中选择 **“负载均衡器公共 IP 地址”** 条目，然后选择 **“下载 RDP 文件”**。

1. 出现提示时，请使用以下凭据登录：

    | 设置 | 数值 | 
    | --- | --- |
    | 用户名 | **Student** |
    | 密码 | **Pa55w.rd1234** |

1. 在 **“az30410a-vm1”** 的远程桌面会话中，在“服务器管理器”窗口中，选择 **“本地服务器”** ，再选择 **“IE 增强的安全配置”** 标签旁的 **“启用”** 链接，然后在 **“IE 增强的安全配置”** 对话框中选择两者的 **“关闭”** 选项。

1. 在与 **“az30410a-vm1”** 的远程桌面会话中，在“服务器管理器”窗口中，选择 **“工具”**，然后在下拉列表中选择 **“Active Directory 管理中心”**

1. 在 **“Active Directory 管理中心”** 中选择 **“adatum（本地）”**，在 **“任务”** 窗格中选择 **“新建”**，然后在级联菜单中选择 **“组织单位”**。

1. 在 **“创建组织单位”** 窗口中，在 **“名称”** 文本框内键入 **“ToSync”**，然后选择 **“确定”**。

1. 双击新创建的 **“ToSync”** 组织单位，使其内容显示在 Active Directory 管理中心控制台的详细信息窗格中。 

1. 在 **“任务”** 窗格中，在 **“ToSync”** 部分，选择 **“新建”**，然后在级联菜单中，选择 **“用户”**。

1. 在 **“创建用户”** 窗口中，使用以下设置新建一个用户帐户（将其他用户保留其现有值），然后选择 **“确定”**：

    | 设置 | 数值 | 
    | --- | --- |
    | 全名 | **aduser1** |
    | 用户 UPN 登录 | **aduser1** |
    | 用户 SamAccountName 登录 | **aduser1** |
    | 密码 | **Pa55w.rd1234** | 
    | 其他密码选项 | **密码永不过期** |


#### 任务 3：安装 Azure AD Connect

1. 在与 **“az30410a-vm1”** 的远程桌面会话中，启动 Internet Explorer，导航到[“Azure 门户”](https://portal.azure.com)，并使用你在前面的练习中创建的 **“az30410-aaduser1”** 用户帐户登录。出现提示时，指定你记录的完整用户名以及 **“Pa55w.rd1234”** 密码。

1. 在 Azure 门户中搜索并选择 **“Azure Active Directory”**，然后在 **“Adatum 实验室 | 概览”** 边栏选项卡上选择 **“Azure AD Connect”**。

1. 在 **“Adatum 实验室 | Azure AD Connect”** 边栏选项卡上，选择 **“下载 Azure AD Connect”** 链接。你将被重定向到 **“Microsoft Azure Active Directory Connect”** 下载页面。

1. 在 **“Microsoft Azure Active Directory Connect”** 下载页面，选择 **“下载”**。

1. 出现提示时，选择 **“运行”** 以启动 **“Microsoft Azure Active Directory Connect”** 向导。

1. 在 **“Microsoft Azure Active Directory Connect”** 向导的 **“欢迎使用 Azure AD Connect”** 页面中，选中复选框 **“我同意许可条款和隐私声明”**，然后选择 **“继续”**。

1. 在 **“Microsoft Azure Active Directory Connect”** 向导的 **“快速设置”** 页面中，选择 **“自定义”** 选项。

1. 在 **“安装所需组件”** 页面中，取消选择所有可选配置选项并选择 **“安装”**。

1. 在 **“用户登录”** 页面中，确保仅启用 **“密码哈希同步”**，然后选择 **“下一步”**。

1. 在 **“连接到 Azure AD”** 页面中，使用你在上一个练习中创建的 **“az30410-aaduser1”** 用户帐户凭据进行身份验证，然后选择 **“下一步”**。 

1. 在 **“连接目录”** 页面上，选择 **“adatum.com”** 林条目右边的 **“添加目录”** 按钮。

1. 在 **“AD 林帐户”** 窗口中，确保选择 **“新建 AD 帐户”** 选项，指定以下凭据，然后选择 **“确定”**：

    | 设置 | 数值 | 
    | --- | --- |
    | 用户名 | **ADATUM\Student** |
    | 密码 | **Pa55w.rd1234** |

1. 返回 **“连接目录”** 页面，确保 **“adatum.com”** 条目显示为已配置目录，然后选择 **“下一步”**

1. 在 **“Azure AD 登录配置”** 页面中，看到警告说明 **“如果 UPN 后缀与已验证域名不匹配，用户将无法使用本地凭据登录 Azure AD”**，启用复选框 **“继续，不将所有 UPN 后缀与已验证域匹配”**，然后选择 **“下一步”**。

    > **注意**：如前所述，这是正常现象，因为你无法验证自定义 Azure AD DNS 域 **adatum.com**。

1. 在 **“域和 OU 筛选”** 页面在选择选项 **“同步选定的域和 OU”**，清除所有复选框，仅选中 **“ToSync”** OU 附近的复选框，然后选择 **“下一步”**。

1. 在 **“唯一标识你的用户”** 页面中，接受默认设置，然后选择 **“下一步”**。

1. 在 **“筛选用户和设备”** 页面中接受默认设置，然后选择 **“下一步”**。

1. 在 **“可选功能”** 页面中接受默认设置，然后选择 **“下一步”**。

1. 在 **“准备配置”** 页面在确保选中 **“配置完成后启动同步过程”** 复选框，然后选择 **“安装”**。

    > **注意**：安装需要约 2 分钟。

1. 请在 **“配置完成”** 页面中查看此信息，然后选择 **“退出”** 关闭 **“Microsoft Azure Active Directory Connect”** 窗口。


#### 任务 4：配置已同步用户帐户的属性

1. 在接入 **az30410a-vm1** 的远程桌面会话的 Azure 门户 Internet Explorer 窗口中，导航到 Adatum Lab Azure AD 租户的 **“用户”-“所有用户”** 边栏选项卡。

1. 在 **“用户” | “所有用户”** 边栏选项卡中，请注意用户对象列表包括 **“aduser1”** 帐户，在 **“源”** 列中有 **“Windows Server AD”**。

    > **注意**：可能需要等待几分钟，然后选择 **“刷新”**，以便显示 **“aduser1”** 用户帐户。

1. 在 **“用户”** | 选择 **“所有用户”** 边栏选项卡中的 **“aduser1”** 条目。

1. 在 **“aduser1” | “个人资料”** 边栏选项卡中，记下用户帐户的全名。

    > **注意**：记录完整用户名。你在下个练习中需要用到它。

1. 在 **“aduser1” | 个人资料”** 边栏选项卡上，在 **“职位信息”** 部分，请注意 **“部门”** 属性未设置。

1. 在接入 **az30410a-vm1** 的远程桌面会话中，切换到 **“Active Directory 管理中心”**，在 **“ToSync”** OU 对象列表中选择 **“aduser1”** 条目，然后在 **“任务”** 窗格中的 **“ToSync”** 部分，选择 **“属性”**。

1. 在 **“aduser1”** 窗口中的 **“组织”** 部分，在 **“部门”** 文本框中键入 **“销售”**，然后选择 **“确定”**。

1. 在接入 **az30410a-vm1** 的远程桌面会话中，启动 **Windows PowerShell**。

1. 从 **“管理员: Windows PowerShell”** 控制台中，运行以下命令，以启动 Azure AD Connect 增量同步：

   ```powershell
   Import-Module -Name 'C:\Program Files\Microsoft Azure AD Sync\Bin\ADSync\ADSync.psd1'

   Start-ADSyncSyncCycle -PolicyType Delta
   ```

1. 切换到显示 **“aduser1 | 个人资料”** 边栏选项卡的 Internet Explorer 窗口，刷新页面，注意 **“部门”** 属性设置为 **“销售”**。

    > **注意**：可能需要等待一分钟，如 **部门** 属性仍未设置请再次刷新页面。

1. 在 **“aduser1 | 个人资料”** 边栏选项卡上，选择 **“编辑”**。

1. 在 **“aduser1 | 个人资料”** 边栏选项卡上，在 **“设置”** 部分的 **“使用位置”** 下拉列表中，选择 **“美国”**，然后选择 **“保存”**。

1. 在 **“aduser1 | 个人资料”** 边栏选项卡上，选择 **“许可证”**。

1. 在 **“aduser1 | 许可证”** 边栏选项卡上，选择 **“+ 分配”**。

1. 在 **“更新许可证分配”** 边栏选项卡上，选择 **“Azure Active Directory Premium P2”** 复选框，然后选择 **“保存”**。


### 练习 3：实现 Azure AD 条件访问
  
本次练习的主要任务如下：

1. 禁用 Azure AD 安全默认值。

1. 创建 Azure AD 条件访问策略

1. 验证 Azure AD 条件访问

1. 删除实验室中部署的 Azure 资源


#### 任务 1：禁用 Azure AD 安全默认值。

1. 在与 **az30410a-vm1** 的远程桌面会话中，在显示 Azure 门户的 Internet Explorer 窗口中，导航到 **“Adatum 实验室”** | Adatum 实验室 Azure AD 租户的 **概述”** 边栏选项卡。

1. 在 **“Adatum 实验室 | 概述”** 边栏选项卡上，在 **“管理”** 部分，选择 **“属性”**。

1. 在 **“Adatum 实验室 | 属性”** 边栏选项卡上，选择页面底部的 **“管理安全默认值”** 链接。

1. 在 **“启用安全默认值”** 边栏选项卡上，将 **“启用安全默认值”** 开关设置为 **“否”**，选中复选框 **“我的组织正在使用条件访问”**，然后选择 **“保存”**。 


#### 任务 2：创建 Azure AD 条件访问策略

1. 在 **“Adatum 实验室 | 属性”** 边栏选项卡上，在 **“管理”** 部分，选择 **“安全”**。

1. 在 **“安全上 | 入门”** 边栏选项卡上，选择 **“条件访问”**。

1. 在 **“条件访问 | 策略”** 边栏选项卡上，选择 **“+ 新建策略”**。

1. 在 **“新建”** 边栏选项卡上，在 **“名称”** 文本框中，键入 **“Azure 门户 MFA 强制”**。 

1. 在 **“新建”** 边栏选项卡上的在 **“作业”** 部分，选择 **“用户和组”**，在 **“包括”** 选项卡上，选择 **“选择用户和组”**，选择 **“用户和组”** 复选框，在 **“选择”** 边栏选项卡上，选择 **“aduser1”**，然后单击 **“选择”** 以确认你的选择。

1. 回到 **“新建”** 边栏选项卡，在 **“作业”** 部分，选择 **“云应用或操作”**，在 **“包括”** 选项卡上，选择 **“选择应用”**，单击 **“选择”**，在 **“选择”** 边栏选项卡上，选择 **“Microsoft Azure 管理”** 复选框，然后单击 **“选择”** 以确认你的选择。

1. 回到 **“新建”** 边栏选项卡，在 **“访问控制”** 部分，选择 **“授权”**，在 **“授权”** 边栏选项卡上，确保选择 **“授权”** 选项，选择 **“需要多重身份验证”**，然后单击 **“选择”** 以确认你的选择。

1. 回到 **“新建”** 边栏选项卡，将 **“启用策略”** 开关设置为 **“开”**，然后选择 **“创建”**。


#### 任务 3：验证 Azure AD 条件访问

1. 在与 **az30410a-vm1** 的远程桌面会话中，启动新的 **“InPrivate 浏览”** Internet Explorer 窗口，并导航到“访问面板应用程序”门户 [https://account.activedirectory.windowsazure.com](https://account.activedirectory.windowsazure.com)。

1. 出现提示时，使用已同步的 **“aduser1”** Azure AD 帐户登录，使用你在上一个练习中记录的完整用户名以及 **“Pa55w.rd1234”** 密码。 

1. 验证你是否可以成功登录到访问面板应用程序门户。 

1. 在同一个浏览器窗口中，导航到 [Azure 门户](https://portal.azure.com)。

1. 请注意，这次你看到的消息是 **“需要更多信息”**。在显示消息的页面中，选择 **“下一步”**。 

1. 此时，你将被重定向到 **“额外安全验证”** 页面，它将引导你逐步完成多重身份验证的配置。

    > **注意**：完成多重身份验证配置是可选的。如果继续，则需要将移动设备指定为身份验证手机或使用它来运行移动应用。


#### 任务 4：删除实验室中部署的 Azure 资源

1. 在与 **“az30410a-vm1”** 的远程桌面会话中，启动 Internet Explorer 并浏览到位于以下位置的适用于 IT 专业人员的 Microsoft Online Services 登录助手 RTW：[https://go.microsoft.com/fwlink/p/?LinkId=286152](https://go.microsoft.com/fwlink/p/?LinkId=286152)。 

1. 在“适用于 IT 专业人员的 Microsoft Online Services 登录助手 RTW”下载页面上，选择 **“下载”**，在 **“选择你想要的下载”** 页面上，选择 **“en\msoidcli_64.msi”**，然后选择 **“下一步”**。 

1. 出现提示时，使用默认选项运行 **“Microsoft Online Services 登录助手安装程序”**。

1. 设置完成后，在与 **“az30410a-vm1”** 的远程桌面会话中，启动 **“Windows PowerShell”** 控制台。

1. 在 **管理员：“Windows PowerShell”** 窗口中，运行以下命令以安装所需的 PowerShell 模块：

   ```powershell
   [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
   Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force
   Install-Module MSOnline -Force
   ```
1. 在 **管理员：“Windows PowerShell”** 窗口中，运行以下命令以向 **“Adatum 实验室”** Azure AD 租户进行身份验证：

   ```powershell
   Connect-MsolService
   ```

1. 当提示你进行身份验证时，请提供 **“az30410-aaduser1”** 用户帐户的凭据。

1. 在 **管理员：“Windows PowerShell”** 窗口中，运行以下命令，禁用 Azure AD Connect 同步：

   ```powershell
   Set-MsolDirSyncEnabled -EnableDirSync $false -Force
   ```

1. 在实验室计算机上显示 Azure 门户的浏览器窗口中，导航到 **“Azure Active Directory Premium P2 - 许可用户”** 边栏选项卡，选择你在本实验室中分配了许可证的用户帐户，选择 **“删除许可证”**，然后在提示确认时选择 **“确定”**。

1. 在 Azure 门户中，导航到 **“用户 - 所有用户”** 边栏选项卡，并确保你在本实验室中创建的所有用户帐户在 **“源”** 列中都有 **“Azure Active Directory”** 条目。 

1. 在 **“用户 - 所有用户”** 边栏选项卡上，选择你在本实验室中创建的每个用户帐户，然后在工具栏中选择 **“删除”**。 

1. 导航到 Adatum 实验室 Azure AD 租户的 **“Adatum 实验室 - 概述”** 边栏选项卡，选择 **“删除租户”**，在 **“删除目录‘Adatum 实验室’”** 边栏选项卡上，选择 **“获取删除 Azure 资源的权限”** 链接，在 Azure Active Directory 的 **“属性”** 边栏选项卡上，将 **“访问 Azure 资源管理”** 设置为 **“是”**，然后选择 **“保存”**。

1. 从 Azure 门户注销，然后重新登录。 

1. 导航回 **“删除目录‘Adatum 实验室’”** 边栏选项卡，然后选择 **“删除”**。

1. 在 **“az30410a-vm1”** 的远程桌面会话中 ，在显示 Azure 门户的浏览器窗口中，在“Cloud Shell”窗格中启动 PowerShell 会话。

1. 在“Cloud Shell”窗格中运行以下命令，以列出你在本练习中创建的资源组：

   ```powershell
   Get-AzResourceGroup -Name 'az30410*'
   ```

    > **注意**：验证输出结果是否仅包含你在本实验室中创建的资源组。在本任务中将删除这个组。

1. 在“Cloud Shell”窗格中运行以下命令，以删除在本实验室中创建的资源组

   ```powershell
   Get-AzResourceGroup -Name 'az30410*' | Remove-AzResourceGroup -Force -AsJob
   ```

1. 关闭“Cloud Shell”窗格。
