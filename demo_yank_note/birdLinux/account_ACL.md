# account manage

::: tip 
- 用户标识符：UID GID
- /etc/passwd
    - UID 0(系统管理员) 1-999(系统账号) 1000~ (可登录账号)
- /etc/shadow
    - EX💯 daemon:*:18677:0:99999:7::: 第三字段(modify time)改为==0==强制用户==首次登录修改密码== 第五字段(密码需要重新修改天数) 
    - root密码忘记，进入单人维护模式，系统会主动给予root权限的bash接口
    - 第三字段改为==0==强制用户==首次登录修改密码==；
- /etc/group
    - 有效用户组和初始用户组
        - 创建账号时添加初始用户组：useradd -g groupName user 
        - 增加有效用户组：usermod -a -G groupName user(或直接group文件中分组后添加用户即可)
        - command： groups (list all support group)  newgrp groupName(change primary group)

:::

::: tip 
- useradd 
    -  EX👨‍🍳 : useradd -u 503247565 -m -d /home/lh247565 -g mrcc -s /bin/csh -c "comment " lh247565     
    - useradd -D 添加用户时的默认值，/etc/default/useradd;
    - SKEL= ==/etc/skel== :用户家目录参考的基目录， 可以设置该用户环境变量，在目录下创建文件，如.bashrc等
    -  /etc/login.defs 定义的uid gid 密码参数设置，邮箱位置
- 批量创建账号后，批量设置密码：echo ${userpwd} | passwd --stdin userName
- userdel  -r  user 连同用户家目录一同删除
- groupadd / groupdel [groupName] 创建及删除组
- /etc/sudoers 添加sudo用户，nopasswd and limit command;

:::

::: tip logon PAM
- 使用外部身份认证系统(NIS/LDAP)； authconfig-tui 图像化接口
    - ==issue==: 用户账号过期，在/etc/security/access.conf 开放group access权限 ==+: mrcc : ALL== 
    - /etc/security/limit.conf (修改完即生效) EX: ==user hard maxlogins 1== 限制每次仅能有一个使用者登录系统
    - 登录产生的错误都会记录在log: seceure and messgages
:::

::: tip 用户信息传递
- w/who 当前登录系统用户； last 显示最近登录系统时间
- mesg y ：开启消息通道；
- wall "xxx messgages" 广播信息给所有用户
- mail -s "Subject ..." account/email < /file
:::






