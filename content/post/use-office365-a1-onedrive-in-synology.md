---
title: "群晖使用 A1 订阅版 Onedrive"
description: 
date: 2024-03-19T11:22:46+08:00
image: 
math: 
hidden: false
comments: true
tags: ["office365", "onedrive", "webdav", "docker", "rclone", "sharepoint"]
categories: ["network"]
draft: true
---

教育版 A1 订阅的 Onedrive 可以用 rclone 以 webdav 方式挂载。 但这不是标准的 webdav 协议，无法在群晖中使用。

群晖的 Cloud Sync 支持同步 Microsoft OneDrive for Business 和 Microsoft SharePoint，但是这个帐号使用 edu 邮箱自己注册的，没有管理员，登陆后报错

```json
{"error":"invalid_client","error_description":"AADSTS650051: Using application 'Cloud Sync' is currently not supported for your organization xxx.edu.cn because it is in an unmanaged state. An administrator needs to claim ownership of the company by DNS validation of xxx.edu.cn before the application Cloud Sync can be provisioned. Trace ID: a4ec2174-8d9f-4281-9f5b-7d782c060601 Correlation ID: c6bdc0ff-5a6c-4597-9060-af9c981610e2 Timestamp: 2024-03-15 02:00:57Z"}
```

## 实现方案
### skleeschulte/basic-to-sharepoint-auth-http-proxy 方案
使用 https://github.com/skleeschulte/basic-to-sharepoint-auth-http-proxy 中转请求，最终以 webdav 协议使用。

### rclone 方案
rclone 也是转为 webdav 服务

`rclone serve webdav remote:path [flags]`

### 方案效果对比

| 应用程序 | basic-to-sharepoint-auth-http-proxy | rclone |
| --- | --- | --- |
| File Station | N | Y |
| Cloud Sync | Y | Y |
| Hyper Backup | Y | Y |

对比后 rclone 方案有明显优势:
1. 支持在 File Station 挂载
2. rclone 是是个活跃项目，basic-to-sharepoint-auth-http-proxy 已经 2 年无更新


## 操作方法
### basic-to-sharepoint-auth-http-proxy 方案
`sudo docker run --name sharepoint-proxy -d -p 13000:3000 -e PROXY_TARGET=https://xxx-my.sharepoint.com/ --restart always skleeschulte/basic-to-passport-auth-http-proxy:v0.1.4`

之后 webdav url 是 http://127.0.0.1:13000/personal/[YOUR-EMAIL]/Documents, 用户名和密码是 onedrive 的信息。

### rclone 方案
rclone 我本地也在用，没用 docker。
#### 1. 配置 rclone 访问 onedrive
教育版 Onedrive 登陆后的地址是 `https://[YOUR-DOMAIN]-my.sharepoint.com/personal/[YOUR-EMAIL]/_layouts/15/onedrive.aspx`, 只需要把 `_layouts/15/onedrive.aspx` 换成 `Documents` 就是 webdav 地址。

`rclone confige - n) New remote - 随便起个名字 - 51 / WebDAV - https://[YOUR-DOMAIN]-my.sharepoint.com/personal/[YOUR-EMAIL]/Documents -  4 / Sharepoint Online, authenticated by Microsoft account - 用户名 - y) Yes, type in my own password - 密码 - 确认密码 - 回车跳过 - y) Yes this is OK (default)`

#### 2. rclone 开启 webadv 服务
以上一步起的名字 1dr-sharepoint 为例, 执行下面命令即可开启 webdav 服务。
`rclone rclone serve webdav 1dr-sharepoint:/ --user user --pass password --addr 192.168.1.101:8080`

webdav url http://192.168.1.101:8080/

rclone 支持开启的服务列表如下:
```bash
  dlna        Serve remote:path over DLNA
  docker      Serve any remote on docker's volume plugin API.
  ftp         Serve remote:path over FTP.
  http        Serve the remote over HTTP.
  nfs         Serve the remote as an NFS mount
  restic      Serve the remote for restic's REST API.
  s3          Serve remote:path over s3.
  sftp        Serve the remote over SFTP.
  webdav      Serve remote:path over WebDAV.
```

#### 3. 群晖 File Station 挂载
File Station - Tools - Remote Connection - Connection Setup - WebDAV
```config
ip: 192.168.1.101
port: 8080
path: /
account name: user
passowrd: pssword
```

## 参考链接
- [DSM Hyper Backup OneDrive 備份教學](https://blog.edmond.work/2023/02/19/dsm-hyper-backup-onedrive-%E5%82%99%E4%BB%BD%E6%95%99%E5%AD%B8/)
- [使群晖 CloudSync 支持同步世纪互联 OneDrive](https://hostloc.com/thread-785905-1-1.html)
- [通过synology的hyper backup备份到microsoft onedrive for business（sharepoint)](https://tongjunsz.gitee.io/2020/10/29/%E9%80%9A%E8%BF%87Synology%E7%9A%84Hyper-Backup%E5%A4%87%E4%BB%BD%E5%88%B0Microsoft-OneDrive-for-Business-SharePoint/)
- [Synology Hyper Backup 備份到 OneDrive](https://blog.ty-wu.com/posts/synohyperbksharepoint/)
- [ WebDAV](https://rclone.org/webdav/)
- [rclone serve webdav](https://rclone.org/commands/rclone_serve_webdav/)
