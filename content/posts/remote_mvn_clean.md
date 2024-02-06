---
title: "远程编译"
date: 2024-02-06T17:00:26+08:00
draft: false
---

今天同事用局域网内闲置电脑搞了个远程编译环境。本地脚本如下，效果很好，编译再也不用卡了
```
#!/usr/bin/env zsh
usage() { echo "Usage: ./rb.sh [-t] [-rf <model>]" 1>&2; exit 1; }
arg="-Dmaven.javadoc.skip=true -Dcheckstyle.skip=true -Dspotbugs.skip=true -Drat.skip=true -Djacoco.skip=true -DskipITs -DskipTests -Prelease"
while getopts "tr:" o; do
    case "${o}" in
        t) arg="-Drat.skip=true -Dcheckstyle.skip=false -T1" ;;
        r) rf="-rf ${OPTARG}" ;;
        *) usage ;;
    esac
done
shift $((OPTIND-1))
rsync -azh --progress --delete --exclude={'**/target','.git','.idea'} "$(pwd)" chuxin@llt-mbp.local:~/dev
rsync -azh --progress ~/.m2/ chuxin@llt-mbp.local:~/.m2/
# shellcheck disable=SC2087
ssh -tt -i /Users/chenchuxin/.ssh/id_rsa chuxin@llt-mbp.local << EOF
cd ~/dev/${PWD##*/}
./mvnw clean install $arg $rf
exit 0
EOF
rsync -azh --progress --include='**/target' chuxin@llt-mbp.local:~/dev/"${PWD##*/}"/ "$(pwd)"
rsync -azh --progress chuxin@llt-mbp.local:~/.m2/ ~/.m2
```
原理其实也很简单，就是使用 rsync 来同步，先将依赖同步过去，编译完后再将文件同步回来
