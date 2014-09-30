未分类笔记
---------------
#### 下载各种软件包
/scripts/feeds update [-a/软件名] 更新软件包
/scripts/feeds install [-a/软件名] 安装软件包


* 启动openwrt前会先执行`package/base-files/files/lib/preinit`里的脚本


* 修改wifi设置:`/package/kernel/mac80211/files/lib/wifi/mac80211.sh`
```bash
config wifi-iface
    option device   radio$devidx
    option network  lan
    option mode     ap
    #自定义SSID为MakeBlaze_MAC后六位
    option ssid     MakeBlaze_$(cat /sys/class/ieee80211/${dev}/macaddress|awk -F ":" '{print $4""$5""$6 }'| tr a-z A-Z) 
    option encryption none
```

* 修改luci设置: `feeds/luci/modules/base/root/etc/config/luci`

```bash
config core main
    option resourcebase '/luci-static/resources'
    #设置默认语言
    option lang 'zh_cn'
    #设置默认主题
    option mediaurlbase '/luci-static/bootstrap'

#设置可选择的语言        
config internal languages
    option zh_cn 'chinese'
    option en 'English'
#设置可选择的主题
config internal themes
    option Bootstrap '/luci-static/bootstrap'
```

* 添加释放内存 `feeds/luci/modules/admin-full/luasrc/controller/admin/index.lua`

```diff
function index()
+    entry({"admin", "Free_Memory"}, call("Free_Memory"), _("释放内存"), 75)
end

function Free_Memory()
    luci.util.exec("echo 3 > /proc/sys/vm/drop_caches")
    luci.http.redirect(luci.dispatcher.build_url("admin", "status", "overview"))
end

```

* 添加重启 `feeds/luci/modules/admin-full/luasrc/controller/admin/index.lua`

```bash
function index()
    entry({"admin", "reboot"}, call("action_reboot"), _("Reboot"), 90)
end
```

* 修改配置文件
`package/base-files/files/etc/` <br>
在这个目录里，你可以把你的备份的路由器导入进来

* 修改路由连接数 `package/base-files/files/etc/sysctl.conf`
`net.netfilter.nf_conntrack_max=65535`

* 修改主机名，设定时区 `package/base-files/files/etc/config/system`

```
config system
    #设置主机名
    option hostname 'MakeBlaze'
    option conloglevel '8'
    option cronloglevel '8'
    #设置时区
    option zonename 'Asia/Shanghai'
    option timezone 'CST-8'

#设置时间服务器
config timeserver 'ntp'
    option enable_server '1'
    list server '210.72.145.44'
    list server 's1a.time.edu.cn'
    list server 's1b.time.edu.cn'
    list server '202.120.2.101'

config led 'usb_led'
    option name 'USB'
    option sysfs 'hg255d:usb'
    option trigger 'usbdev'
    option dev '1-1'
    option interval '50'

config led 'wlan_led'
    option name 'WLAN'
    option sysfs 'hg255d:wlan'
    option trigger 'netdev'
    option dev 'ra0'
    option mode 'link tx'

config led 'internet_led'
    option name 'INTERNET'
    option sysfs 'hg255d:internet'
    option trigger 'netdev'
    option dev 'eth0.2'
    option mode 'tx rx'
```

* 添加将自己开发的程序

在package下建立一个文件夹，以自己的项目命名，里边包括一个files文件夹和一个Makefile文件

```bash
目录结构
<ccnu>
├Makefile
├<files>
│  ├<etc>
│  │  ├<config>
│  │  │  └ccnu
│  │  ├<init.d>
│  │  │  └ccnu
│  ├<usr>
│  │  ├<lib>
│  │  │  ├<lua>
│  │  │  │  ├<luci>
│  │  │  │  │  ├<controller>
│  │  │  │  │  │  └ccnu.lua
│  │  │  │  │  ├<model>
│  │  │  │  │  │  ├<cbi>
│  │  │  │  │  │  │  └ccnu.lua
│  │  ├<sbin>
│  │  │  └ccnu
```

`package/ccnu/Makefile`

```bash
include $(TOPDIR)/rules.mk

PKG_NAME:=ccnu
PKG_VERSION:=2.0.1        
PKG_RELEASE:=1
PKG_BUILD_DIR := $(BUILD_DIR)/$(PKG_NAME)

include $(INCLUDE_DIR)/package.mk

define Package/ccnu
  SECTION:=utils
  CATEGORY:=Utilities
  SUBMENU:=H3C inode
  TITLE:= iNode Compatible Client for CCNU
  DEPENDS:=+libc +libgcc 
  MAINTAINER:= RiceLyn <fjkfwz@gmail.com>
endef

define Package/ccnu/description
    iNode Compatible Client for CCNU
endef

define Build/Prepare
endef

define Build/Configure
endef

define Build/Compile
endef

define Package/ccnu/install
    $(CP) ./files/* $(1)/
endef

$(eval $(call BuildPackage,ccnu))
```

之后./scripts/feeds install -a,ccnu就会出现在Utilities -> H3C inode里啦
这里先做一个简单的程序，以后会写文章详细介绍。

* 开启POWER LED状态指示灯

```bash
get_status_led() {
    case $(ramips_board_name) in
+    hg255d)
+        status_led="hg255d:power"
+        ;;
```

* 添加自动挂载脚本

`/package/base-files/files/etc/gotplug/block/30-block_mount`

```bash
#!/bin/sh

#---------------------------------------------------------------------
# Filename:    30-block_mount
# Revision:    0.5.1
# Data:        Jul. 7, 2014
# Email:       kevinyu@vip.qq.com
# Contacts:    QQ/TM 389191 or mail address above
# Licensing:   General Public License v2
# Description: Created for automatic mount block devices on OpenWRT
# Usage:       Put this script into directory /etc/hotplug.d/block
#---------------------------------------------------------------------

## Global settings of auto-mount script ##

# Turn on|off auto-mount function for disk volumes
VOL_ENABLED=1  # <0|1  0-Disable, 1-Enable>

# Turn on|off auto-mount function for sawp partition  
SWAP_ENABLED=1  # <0|1  0-Disable, 1-Enable>

# Set which method you want to name the mounted volumes
USE_VLABEL=1  # <0|1  0-Use device name, 1-Use volume label>  Eg. /mnt/sda1 | /mnt/DataDisk
#
# Note that if you choose name mount point by volume label but the volume's label is empty,
# the volume will be named to 'volume_$PARTUUID', such as the /mnt/volume_31173116-01
# Also, if exist more than one device which have the same volume label, it will be renamed 
# as the original name + sequence number, such as the /mnt/DataDisk_n (n=1,2,3..9)

# Show filesystem type as the prefix of mount point, requires set USE_VLABEL=1
FSTYPE_IN_VLABEL=1  # <0|1  0-Disable, 1-Enable>  Eg. /mnt/[ntfs]-DataDisk

# Set access the mounted device with read-write or read-only permission
FS_READONLY=0  # <0|1  0-Read Write, 1-Read Only>

# Exclusion list of auto-mount function
# Allows mixing the device name, volume label and UUID, each item must be separated by space(s)
# Eg. EXCLUDE_LIST="sda1 DataDisk f32b2ea6-4d42-43a5-bcec-7d818a163d07" 
EXCLUDE_LIST=""  # Default: empty string

# Specify a directory for mounts
MOUNTS_PATH="/mnt"  # Default: /mnt , don't need include the trailing slash (/)

# Define the add-ons scripts directory for automatic execute
ADDSH_PATH="/etc/hotplug.d/user"  # Script naming rule: [0-9][0-9]-xxxxx, Eg. 20-backup-sdcard

## End of global settings ##

# Purge the directory MOUNTS_PATH
purge_mnts()
{
local i RETVAL
[ -d $MOUNTS_PATH ] || return 1
for i in $MOUNTS_PATH/* ; do
    if [ -d $i ] ; then
        if [ -z "`ls -A $i 2>&-`" ] ; then
            `grep -qs "$i" /proc/mounts` || { rm -rf $i 2>/dev/null; RETVAL=$?; }
            if [ $RETVAL -eq 0 ] ; then
                logger -t Auto-Mount "[Notice] Unused mount point $i was removed"
                [ "$CLI" = "yes" ] && echo "[Notice] Unused mount point '$i' was removed"
            else
                logger -t Auto-Mount "[Error] Remove unused mount point $i failed"
                [ "$CLI" = "yes" ] && echo "[Error] Remove unused mount point $i failed"
            fi
        fi
    fi
done
}

# Unmount all block device
umountall()
{
local i MOUNTS
MOUNTS=`grep -os '^/dev/sd[a-z][1-9]' /proc/mounts`
for i in $MOUNTS ; do
    umount -f $i
    [ $? -eq 0 ] &&    echo "[Notice] Device $i unmounted" | tee /proc/self/fd/2 | logger -t Auto-Mount
done
purge_mnts
}

# Detect blkid exists
BLKID=`which blkid`
if [ $? -ne 0 ] ; then
    logger -t Auto-Mount "[Error] Unable to mount block devices automatically because 'blkid' does not exist"
    exit 1
fi

MOUNTS_PATH=${MOUNTS_PATH%/}
[ -z "$ACTION" ] && ACTION=${1##*-}
[ -n "$DEVICENAME" ] && DEV=$DEVICENAME || DEV=$2
logger -t Auto-Mount "[Debug] action='$ACTION' devicename='$DEV'"
case "$ACTION" in
    add)
        [ -z "$DEV" ] && { logger -t Auto-Mount "[Error] Missing required arguments: \$devicename"; exit 1; }
        if [ -n "$EXCLUDE_LIST" ] ; then
            UUID=`$BLKID -s UUID /dev/$DEV | awk -F '=' '{print $NF}' | tr -d '"| '`
            VLABEL=`$BLKID -d -s LABEL /dev/$DEV | awk -F '=' '{print $NF}' | tr -d '"| '`
            for i in $EXCLUDE_LIST ; do
                ignore=0
                if `echo "$i" | grep -qs '^sd[a-z][1-9]$'` ; then  # is device name
                    [ $i = $DEV ] && ignore=1
                elif `echo "$i" | grep -Eiqs '^[0-9a-f]{8}(-[0-9a-f]{4}){3}-[0-9a-f]{12}$'` ; then  # is UUID
                    [ $i = $UUID ] && ignore=1
                else  # as volume label
                    [ $i = $VLABEL ] && ignore=1
                fi
                [ $ignore -eq 1 ] && { logger -t Auto-Mount "[Notice] Device $DEV is ignored because it in exclusion list"; exit 0; }
            done
        fi
        if `echo "$DEV" | grep -qs '^sd[a-z][1-9]$'` ; then
            [ -z "$VLABEL" ] && VLABEL=`$BLKID -d -s LABEL /dev/$DEV | awk -F '=' '{print $NF}' | tr -d '"| '`
            [ -z "$VLABEL" ] && VLABEL="unknown"
            FSTYPE=`$BLKID -s TYPE /dev/$DEV | awk -F '=' '{print $NF}' | tr -d '"| '`
            logger -t Auto-Mount "[Debug] devicename='$DEV' volume_label='$VLABEL' fstype='$FSTYPE'"
            MCMD=""  # set default value
            [ $FS_READONLY -eq 1 ] && acs="ro" || acs="rw"
            case "$FSTYPE" in
                ext[234])
                    if `grep -vs '^nodev' /proc/filesystems | grep -qs "$FSTYPE"` ; then
                        MCMD="mount -t $FSTYPE -o ${acs},noatime"
                    fi
                    ;;
                xfs|reiserfs)
                    if `grep -vs '^nodev' /proc/filesystems | grep -qs "$FSTYPE"` ; then
                        MCMD="mount -t $FSTYPE -o ${acs},noatime"
                    fi
                    ;;
                iso9660|udf)
                    if `grep -vs '^nodev' /proc/filesystems | grep -qs "$FSTYPE"` ; then
                        MCMD="mount -t $FSTYPE -o ro"
                    fi                    ;;
                vfat|msdos)
                    if `grep -vs '^nodev' /proc/filesystems | grep -qs "$FSTYPE"` ; then
                        MCMD="mount -t $FSTYPE -o ${acs},codepage=936,umask=000"  #iocharset=utf8
                    fi
                    ;;
                exfat)
                    if `grep -Eqs '^[[:space:]]+fuseblk$' /proc/filesystems` && `which mount.exfat >/dev/null` ; then
                        MCMD="mount.exfat -o ${acs},iocharset=cp936,umask=000"
                    fi
                    ;;
                ntfs)
                    if `grep -Eqs '^[[:space:]]+fuseblk$' /proc/filesystems` && `which mount.ntfs-3g >/dev/null` ; then
                        MCMD="mount -t ntfs-3g -o ${acs},noatime,big_writes,async,umask=000"
                    fi
                    ;;
                swap)
                    [ $SWAP_ENABLED -ne 1 ] && exit 0
                    if [ `grep -vs '^Filename' /proc/swaps | wc -l` -eq 0 ] ; then
                        MCMD="swapon /dev/$DEV"
                    fi
                    ;;
                *)
                    MCMD=""
                    ;;
            esac
            [ $VOL_ENABLED -ne 1 ] && exit 0
            if [ $FSTYPE = "swap" ] ; then
                MOPT=""
            elif [ $FSTYPE = "iso9660" -o $FSTYPE = "udf" ] ; then
                CDROM="cdrom"
                if `grep -qs "$MOUNTS_PATH/$CDROM " /proc/mounts` ; then
                    SUFFIX=1
                    while `grep -qs "$MOUNTS_PATH/$CDROM " /proc/mounts` ; do
                        [ $SUFFIX -lt 10 ] && CDROM="${CDROM%_*}_$SUFFIX" || { CDROM="${CDROM%_*}_99"; break; }
                        let SUFFIX++
                    done
                fi
                MPOINT=$CDROM
            else
                if [ $USE_VLABEL -eq 1 ] ; then
                    if [ $VLABEL = "unknown" ] ; then
                        PARTUUID=`$BLKID -s PARTUUID /dev/$DEV | awk -F '=' '{print $NF}' | tr -d '"| '`
                        [ -z "$PARTUUID" ] && PARTUUID=`cat /proc/sys/kernel/random/uuid | cut -f1 -d'-'`
                        VLABEL="volume_${PARTUUID}"
                    elif `grep -qis "$MOUNTS_PATH/$VLABEL " /proc/mounts` ; then
                        SUFFIX=1
                        while `grep -qis "$MOUNTS_PATH/$VLABEL " /proc/mounts` ; do
                            if [ $SUFFIX -lt 10 ] ; then
                                VLABEL="$(echo "$VLABEL" | sed 's/_[0-9]\+$//')_$SUFFIX"
                                let SUFFIX++
                            else
                                VLABEL="$(echo "$VLABEL" | sed 's/_[0-9]\+$//')_99"
                                break
                            fi
                        done
                    fi
                    [ $FSTYPE_IN_VLABEL -eq 1 ] && MPOINT="[${FSTYPE}]-${VLABEL}" || MPOINT=$VLABEL
                else
                    MPOINT=$DEV
                fi
                MOPT="/dev/$DEV $MOUNTS_PATH/$MPOINT"
            fi
            if [ -n "$MCMD" ] ; then
                if ! `grep -qs '/dev/$DEV' /proc/mounts` ; then
                    [ $FSTYPE = "swap" ] || { [ -d $MOUNTS_PATH/$MPOINT ] || mkdir -p $MOUNTS_PATH/$MPOINT; }
                    logger -t Auto-Mount "[Debug] MountCMD='$MCMD $MOPT'"
                    # echo "[Debug] MountCMD='$MCMD $MOPT'" >> /tmp/auto-mount-debug
                    $MCMD $MOPT
                    RETVAL=$?
                    if [ $FSTYPE = "swap" ] ; then
                        if [ $RETVAL -eq 0 ] ; then
                            logger -t Auto-Mount "[Notice] Swap device /dev/$DEV turned on" 
                        fi
                    else
                        if [ $RETVAL -eq 0 ] ; then
                            logger -t Auto-Mount "[Notice] Block device /dev/$DEV mounted on $MOUNTS_PATH/$MPOINT" 
                        else
                            logger -t Auto-Mount "[Warning] Re-try mount the block device /dev/$DEV after 2 seconds..."
                            sleep 2
                            $MCMD $MOPT
                            if [ $? -eq 0 ] ; then
                                logger -t Auto-Mount "[Notice] Block device /dev/$DEV mounted on $MOUNTS_PATH/$MPOINT" 
                            else
                                rm -rf $MOUNTS_PATH/$MPOINT 2>/dev/null
                                logger -t Auto-Mount "[Error] Mount block device /dev/$DEV encounters an error"
                                exit 1
                            fi
                        fi
                    fi
                fi
            else
                if [ $FSTYPE = "swap" ] ; then
                    logger -t Auto-Mount "[Notice] Swap device /dev/$DEV was ignored"
                else
                    logger -t Auto-Mount "[Error] Unsupported filesystem '$FSTYPE' on /dev/$DEV"
                fi
            fi
            unset MCMD
        fi
        purge_mnts
        ;;
    remove)
        [ -z "$DEV" ] && { logger -t Auto-Mount "[Error] Missing required arguments: \$devicename"; exit 1; }
        if `echo "$DEV" | grep -qs '^sd[a-z][1-9]$'` ; then
            if `grep -qs '/dev/$DEV' /proc/mounts` ; then
                MPATH=`awk -F ' ' '/\/dev\/'"$DEV"'/ {print $2}' /proc/mounts 2>/dev/null`
                umount /dev/$DEV
                [ $? -eq 0 ] &&    logger -t Auto-Mount "[Notice] Block device $DEV unmounted"
                [ -d "$MPATH" ] && rm -rf $MPATH 2>/dev/null
            fi
            if `grep -qs '/dev/$DEV' /proc/swaps` ; then
                swapoff $DEV
                [ $? -eq 0 ] &&    logger -t Auto-Mount "[Notice] Swap device $DEV turned off"                 
            fi            
        fi
        purge_mnts
        ;;
    umountall)
        CLI="yes"
        umountall
        ;;
    purge)
        CLI="yes"
        purge_mnts
        ;;
    *)
        echo "[Error] ** Invalid action arguments: $ACTION" | tee /proc/self/fd/2 | logger -t Auto-Mount
        ;;
esac

## Execute the add-ons script
# ADDSH_PATH="$(cd "$(dirname "$0")"; cd ..; pwd)/user"
if [ -n "`ls -A $ADDSH_PATH 2>&-`" ]; then
    i=0
    for addsh in ${ADDSH_PATH}/[0-9][0-9]-* ; do
        [ $i -eq 0 ] && logger -t Auto-Mount "[Notice] -- Execute additional scripts ..."
        logger -t Hotplug-Debug "-- Exec script - $addsh"
        sh $addsh $ACTION $DEV 
        let i++
    done
fi
# Script EOF
```

linux内核部分相关在:

`build_dir/target-mipsel_24kec+dsp_uClibc-0.9.33.2/linux-ramips_mt7620n/linux-3.14.18/arch/mips/kernel`

Q:卡在starting kernel ... 怎么办? <br>
A:有个比printk更早的函数early_printk, 提前它的初始化,在 内核/kernel/setup.c下

HC5661编译
------------------
1.网络端口<br>
`./target/linux/ramips/base-files/etc/board.d/02_network`里加入
`local board="HC5661"`

在y1 y1s前加入HC5661 |\ --> 这个文件是配置网络端口的, 之前lan4用不了就是因为这个,这个文件格式类似case语句

2.LED设置修改<br>
`/trunk/target/linux/ramips/base-files/etc/board.d/01_leds` <br>
`/trunk/target/linux/ramips/base-files/etc/diag.sh`

语法像case, XXX)表示一个新配置,在此之前如果是 YYY |\则表示和XXX共用一套配置

`/trunk/target/linux/ramips/base-files/lib/ramips.sh`是新建一个设备用的, 语法是

```
*"HiWiFi 1S")   
name="HC5661"     
;;
```

`/trunk/target/linux/ramips/base-files/lib/upgrade/platform.sh` 用来识别是否同一CPU的固件

编译bin用的Makefile <br>
`/trunk/target/linux/ramips/image/Makefile` 在相应平台的Profile里仿照修改

系统相关配置,含时区,led等等<br>
`/trunk/package/base-files/files/etc/config/system`


目录结构
------------------
## /packages/
这里存的是除内核外的各种脚本和软件(比如busybox之类), 这些软件被分类在该目录的各个分类中, 由各目录的 *Makefile* 最终决定, 比如 <br>
*/etc/hotplug.d/10-mount* 的最初来自于 */packages/system/fstools/Makefile* 中的

```
$(INSTALL_DATA) ./files/mount.hotplug $(1)/etc/hotplug.d/block/10-mount
```

如果想在 */etc/hotplug.d/* 里加入其他脚本,如 *20-mmcmount* 则可以加入

```diff
+ $(INSTALL_DATA) ./files/mmcmount $(1)/etc/hotplug.d/block/20-mmcmount
```


Linux相关笔记
------------------
#### Starting kernel 后无输出信息
在内核能输出信息前有一个更早的信息的函数*early_printk*