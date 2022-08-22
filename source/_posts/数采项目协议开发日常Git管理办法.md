title: 数采项目协议开发组日常Git管理办法
tags: [树根,数采,Git]
categories: [项目]

date: 2023-10-19 17:36:00
---
### 概览

![content](https://api.onedrive.com/v1.0/shares/s!AuARQ03xNIEng_Z03X9gHM0XRTYtYQ/root/content)

### 日常流程

#### 常规流程

1. 采用分支开发模式，以`master`为开发主分支，每个开发者签出自己的开发分支（例如：`feat-a`）进行开发；
2. `新功能`及`优化项`均在`feat-a`上开发，开发完成后，使用`rebase(变基)`方式将`feat-a`合并到`master`，解决相关冲突，并将个人分支推送到远程仓库分支`remote/feat-a`；
3. 在Git-Lab提交从源分支`remote/feat-a`到目标分支`remote/master`的合并请求； 

#### Bug修复

4. 假设发现`release-1.6`上报告了一个bug，从`release-1.6`拉取个人分支`fix-a-release-1.6`，进行修复，**版本号+1**，并按“**fix:a | ~ ~ ~**”的消息格式提交代码；
5. 审查`master`上是否存在此bug。若存在，拉取个人分支`fix-a-master`，将**步骤4**中所作提交摘取到此分支，或进行手动修复并将版本号+1，同样按“**fix:a | ~ ~ ~**”的消息格式提交代码；
6. 分别从两个个人分支提起到对应主分支的`MR`，审核人员检查到`fix`标注的合并请求时，需要注意是否同时包括`master`和`release`的请求，以时刻保持代码一致。若存在问题，附上评论并打回即可；
7. 若**步骤5**发现master上已不存在该bug（可能某次常规迭代已经修复了），则只要提release分支修复的合并申请，并在`MR`上附上简要说明即可；

#### 优化

8. 若常规code review发现优化项目，当作日常功能开发即可；
9. 若是现场报告的问题对应的优化项，当作bug进行修复；
