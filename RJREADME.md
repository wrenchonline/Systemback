# 榕基编译注意事项

### 如果减少代码开发，那么有些事情在备份系统前需要提前做。

1. 设置grub超时设置,打开/etc/default/grub

```shell
sudo vim /etc/default/grub
```
寻找 GRUB_TIMEOUT 这个字段,然后设置成 0（一般系统默认是0）, -1 是永久显示  其他数字就是页面超时退出。

2. 新建sh文件
```shell
touch set_grub_password.sh
```

在文件中输入shell代码（有可能不完整，后续需要一直修改完善）

```shell
#!/bin/bash

username="root"  # 替换为您的用户名

# 这个密码可以提前生成 sudo grub-mkpasswd-pbkdf2
hashed_password="grub.pbkdf2.sha512.10000.C370C676CCB3C6A568228780933C3B9976F9378A26760406F0AA4DFF6CEBE906B055B360EE2205191F562E8F4E794F640BE5D36D2EC739ED52A9403F479A1DC7.E190B4BCCA18868E796F22F4516C8DD731F7139AFC247495CFB78C21B883BB371A28BB05767A5A3972E240093C9833D6140BBF4AEA2D667207B12D24275CA7A8"

custom_config_path="/etc/grub.d/40_custom"

echo "set superusers=\"$username\"" >> "$custom_config_path"

echo "password_pbkdf2 $username $hashed_password" >> "$custom_config_path"


echo "GRUB random password set successfully"

sed -i 's/CLASS="--class gnu-linux --class gnu --class os"/CLASS="--class gnu-linux --class gnu --class os --unrestricted"/' /etc/grub.d/10_linux

echo "在 /etc/grub.d/10_linux 文件中添加了 --unrestricted 标志"
update-grub

```

之后把这个文件放入/usr/bin/set_grub_password.sh

```shell
cp set_grub_password.sh /usr/bin/set_grub_password.sh
sudo chmod +x /usr/bin/set_grub_password.sh
```

原因如下我在 [systemback::systemcopy()](systemback/systemback/systemback.cpp#L2537) 函数中执行这一行:
```c++
sb::exec("chroot /.sbsystemcopy sh /usr/bin/set_grub_password.sh");
```
所以这个路径拷贝是必须要做的。


这个代码的目的就是设置grub密码和保护菜单注意这段代码需要优化，很可能匹配不对导致失败，实际上就是在`CLASS="(系统内容)"`上加个 `--unrestricted`,意思是grub只保护菜单编辑，不保护菜单启动

```shell
sed -i 's/CLASS="--class gnu-linux --class gnu --class os"/CLASS="--class gnu-linux --class gnu --class os --unrestricted"/' /etc/grub.d/10_linux
```

2. ubuntu禁用快捷键
```shell
gsettings set org.gnome.desktop.wm.keybindings switch-to-workspace-up []
```





# 源代码编译

1. 下载依赖包

```shell
sudo apt-get install build-essential debhelper devscripts libblkid-dev libmount-dev libncursesw5-dev libparted-dev qtbase5-dev qttools5-dev-tools git-all gnupg
```

2. 进入目录
```shell
cd systemback
```

3. 编译deb

```shell
debuild -us -uc
```

4. 之后安装软件
```shell
dpkg -i ../*.deb
```

之后就可以进行livecd生成了。
源码构建参考 [wiki page](https://github.com/MaranBr/Systemback/wiki/Build-from-Source)



# usb启动盘刷入shell命令
```shell
dd  if=(iso文件) of=/dev/sdX 
```


# 遇到错误解决方案

1. 如果`livecd`创建失败，提示设备文件进行严重变化，用控制台命令打开systemback查看错误，发现一般是snap的火狐浏览器导致的，直接snap卸载火狐浏览器即可。

2. 如果`debuild -us -uc`遇到错误: `no dependency information found for  xxxxx` 但是库存在，代表该库是系统版本里的，而不是本地安装的，没有相关的information，所以报上面错误
解决方案 `debian/rules`中增加下面配置
`dh_shlibdeps --dpkg-shlibdeps-params=--ignore-missing-info -a`
而 我发现在这个软件的配置 `binary-arch: build-arch` 存在 `dh_shlibdeps -a`
我修改成 `dh_shlibdeps --dpkg-shlibdeps-params=--ignore-missing-info -a`,成功解决。

3. 遇到源码格式错误问题 `' is invalid 错误: source package format '3.0 (native) `
进入 systemback\debian\source\format  `3.0 (native)` 不能留有空格

4. ubuntu `22.04.3` TLS 编译有错误，可以在 `20.04` TLS 编译或者`22.10`编译在拉过来，目测是readelf这个软件包版本有问题,等待修复。