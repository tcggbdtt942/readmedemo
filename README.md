# 数据同步

[![Project of Huaruiinfo](https://img.shields.io/badge/project%20of%20company-huaruiinfo-blue.svg)](http://www.huaruiinfo.com/)

此项目是将基础数据库中的数据操作, 同步至各个站点的sybase数据库中. 并且把文章, 附件, 图片等同步至相应站点的FTP目录中

## 使用第三方库

| 第三方库 | 名称 | 备注 |
|-|-|-|
| 日志工具 | NLog | [config配置](https://nlog-project.org/config/?tab=layout-renderers) |
| API文档工具 | Swagger | [微软配置文档](https://docs.microsoft.com/zh-cn/aspnet/core/tutorials/getting-started-with-swashbuckle?view=aspnetcore-2.2&tabs=visual-studio) , [API文档地址](http://192.168.0.45:2000/index.html)|
| FTP工具库 | FluentFTP | [github地址](https://github.com/robinrodricks/FluentFTP) |
| 配置中心| Apollo | [配置地址](http://192.168.0.80:8070/) , AppId : `DBASE` |
| 分布式事务解决方案| DotNetCore. CAP| [github地址](https://github.com/dotnetcore/CAP) , [仪表板面板](http://192.168.0.45:2000/cap)|
|消息队列服务|RabbitMQ|[管理地址](http://192.168.0.180:15672/)|
| Sybase数据库工具库| AdoNetCore. AseClient | [github地址](https://github.com/DataAction/AdoNetCore.AseClient) |

## 项目说明

### 1. 接口调用

调用 `SyncController` 中的接口

``` 
/// <summary>
/// 指数同步
/// </summary>
public ActionResult<ResultEntity> CompositeIndexSynch(List<ClientSynchPublish> caploglist)

...

/// <summary>
/// 行情FTP同步
/// </summary>
public ActionResult<ResultEntity> FtpSynch(ClientSynchPublish caplog)
```

### 2. 同步数据发布

服务 `DBaseService` 中的 `CAP` 发布

``` 
/// <summary>
/// CAP发布 - 文章
/// </summary>
/// <param name="pub"></param>
/// <returns></returns>
public string CAPArticlePublish(ClientSynchPublish pub)
{
    string SiteName = "";//需要指定同步的站点名称合集 
    if (string.IsNullOrEmpty(pub.Record))
    {
        return $"{ResultEntity.Failure} : 记录内容为空,同步失败,ID:{pub.Id}";
    }
    foreach (var item in JsonConvert.DeserializeObject<Pub_ArticleDto>(pub.Record).ShareList)
    {
        SiteName += $"{item.Site_ID}.";
    }
    _capPublisher.Publish(EventConstants.CREATE_ARTICLE_ROUTER.Replace("sites.", SiteName), pub);
    return SetCapLog2Pub(pub);//更新发布记录状态
}
...
```

### 3. 数据订阅消费

在 `Subscribe_XXXService` 中订阅消费

``` 
/// <summary>
/// 文章 CCF
/// </summary>
/// <param name="caplog"></param>
/// <returns></returns>
[CapSubscribe(EventConstants.CREATE_ARTICLE_CCF, Group = EventConstants.GROUP_ARTICLE_CCF)]
public void GetMapping(ClientSynchPublish caplog)
{
    _synchArticleService.GetMappingThenSynch(caplog, Site.CCF);
}
```

然后在 `Synch_XXXService` 的 `GetMappingThenSynch` 方法中处理同步数据

## 数据库说明

> 数据库地址 `192.168.0.80` 

### 数据库:dbase

| 表名称                                | 说明                               |
|--------------------------------------|------------------------------------|
| [Cap].[ClientSynchPublish]           | 发布记录表                          |
| [Cap].[ClientSynchPublish_File]      | 发布记录子表 - 记录包含的文件表(FTP同步) |
| [dbo].[TA_Index_Composite]           | **指数**                           |
| [dbo].[TM_Custom_Index_Mapping]      | 指数配置                            |
| [dbo].[TM_Custom_Index_GraphMapping] | 曲线配置                            |
| [dbo].[TA_PriceInfo]                 | **行情**                           |
| [dbo].[TM_GraphMapping]              | 行情 曲线配置                        |
| [dbo].[TM_FieldMapping]              | (行情同步)                          |
| [dbo].[TP_App_Property]              | (行情同步)                          |
| [dbo].[TP_Product]                   | 产品列表(行情同步)                   |
| [dbo].[TA_News_Info]                 | **文章 数据**                       |
| [dbo].[TA_News_Share]                | 文章 共享                           |
| [dbo].[TM_Custom_NewsInfo_Mapping]   | 文章配置                            |
| [dbo].[TM_News_Share_Field]          | 文章配置                            |
| [dbo].[TA_Product_App_Txt]           | **行情FTP**                        |
| [dbo].[TP_Dict]                      | 数据字典                            |

<br/>

### 数据库:dbase_synch

| 表名称                  | 说明                      |
|------------------------|---------------------------|
| [Cap].[Published]      | CAP发布表(CAP自动生成, 修改) |
| [Cap].[Received]       | CAP订阅表(CAP自动生成, 修改) |
| [dbo].[CAP_Data_Logs]  | 数据的同步日志表            |
| [dbo].[CAP_Ftp_Logs]   | FTP的同步日志表             |
| [dbo].[Ftp_Conns]      | FTP连接配置表              |
| [dbo].[Sys_UserNotice] | 同步异常邮件提醒表           |

## 日志记录

### 日志查看

基础数据库 -> 系统管理 -> 日志管理

* CAP-发布日志      `[Cap].[ClientSynchPublish]`
* 数据-同步日志     `[dbo].[CAP_Data_Logs] `
* FTP-同步日志      `[dbo].[CAP_Ftp_Logs] `


![日志管理](README/README_IMG_1.png '日志管理')

