## GitLab触发Jenkins构建job

- Jenkins上集成GitLab，当有人push code到GitLab上时，GitLab会触发Jenkins进行相应Job的构建。
- 构建触发器(webhook)，俗名钩子，实际上是一个 HTTP 回调,其用于在开发人员向 gitlab 提交代码后能够触发 jenkins 自动执行代码构建操作。

### 1.  jenkins安装插件

- 系统管理  ==》插件管理  ==》 安装 GitLab Plugin 、Gitlab Hook Plugin 和 Gitlab Authentication。

### 2. Gitlab 加公钥

- 在Jenkins的服务器上，用**Jenkins**用户或者**root**用户生成ssh密钥对

  > ssh-keygen

- 然后将文件id_rsa.pub里面的内容copy到GitLab上, GitLab ==》 setting ==》 SSH Keys

![1.png](https://github.com/Lqlixl/linux/blob/master/Basic/Picture/Jenkins/githook/1.png?raw=true)



### 3. 更改 Jenkins 配置

- Jenkins系统管理  ==》 全局安全配置

- 授权策略

  - 认证改为"登录用户可以做任何事情"

- 关闭跨站请求伪造保护

  ![4.png](https://github.com/Lqlixl/linux/blob/master/Basic/Picture/Jenkins/githook/4.png?raw=true)

### 4.  配置Job触发

- 在 Jenkins Job里的**配置**里面，勾选Build when a change is pushed to GitLab. GitLab CI Service URL: http://192.168.1.1/project/WEBpcstudent，具体里面的配置都采用默认的。
- 记下这里的 URL 后面需要使用
- 然后点开高级

![6.png](https://github.com/Lqlixl/linux/blob/master/Basic/Picture/Jenkins/githook/6.png?raw=true)

- 设置触发 branches 可以写多个，也可以设置所有分支
- 生成token值 ，在后面gitlab需要配置，生成后为了安全复制出来然后，然后清理。

![7.png](https://github.com/Lqlixl/linux/blob/master/Basic/Picture/Jenkins/githook/7.png?raw=true)

### 5. 配置Gitlab 

- 打开gitlab ,并且找到刚刚在Jenkins配置的 **job**对应的项目地址，
- 点开 setting ==》 Integrations 

![9.png](https://github.com/Lqlixl/linux/blob/master/Basic/Picture/Jenkins/githook/9.png?raw=true)

- 复制前面在Jenkins配置里记下的 URL 和 Token值

![10.png](https://github.com/Lqlixl/linux/blob/master/Basic/Picture/Jenkins/githook/10.png?raw=true)

- 添加 Add webhook

- 点击测试

![12.png](https://github.com/Lqlixl/linux/blob/master/Basic/Picture/Jenkins/githook/12.png?raw=true)

- 测试返回200
- 使用git push ,返回Hook executed successfully: HTTP 200就表示成功，同时Jenkins job也会build起来

![13.png](https://github.com/Lqlixl/linux/blob/master/Basic/Picture/Jenkins/githook/13.png?raw=true)

### 6. 注意事项

- test的时候有可能返回的是Hook executed successfully but returned HTTP 403

  这是没有权限，需要把Jenkins ==》系统管理  ==》 系统设置，找到GitLab配置，Enable authentication  去掉勾选。

  ![14.png](https://github.com/Lqlixl/linux/blob/master/Basic/Picture/Jenkins/githook/14.png?raw=true)



- 设置URL

![15.png](https://github.com/Lqlixl/linux/blob/master/Basic/Picture/Jenkins/githook/15.png?raw=true)