---
title: MTK 9.0 平台添加一个分区
date: 2019-09-27 21:19:25
tags:
    - Android 9.0
    - Mtk
    - partition
    - A/B system
---

最近要在 MTK Android P 的平台, 添加一个 A/B 分区. 这其中踩了很多的坑. 以下是在添加新的分区当中遇到的问题进行的记录.
<!-- more -->

# MTK 9.0 平台添加一个分区
@(aosp/Android)

最近要在 MTK Android P 的平台, 添加一个 A/B 分区. 这其中踩了很多的坑. 以下是在添加新的分区当中遇到的问题进行的记录.

##修改操作

添加一个AB分区, 并且添加相应的 image 文件
涉及到的文件. 已经设计的到的问题的总结:
```
build/core/Makefile  // ② 命令行之前需要有一个TAB, ⑧编译出现了failed with exit 4,  未知原因； ③导致生成的image文件出现错误；   ④生成了相关的image 但是没有放在 target_files_package下的 IMAGES 文件夹下
build/core/config.mk
build/core/config.mk
build/core/envsetup.mk
build/core/main.mk
build/tools/releasetools/add_img_to_target_files.py  //   ④add_img_to_target_files 出现无法找到相应的 image文件, 和makefile其中一个是同一个问题
build/tools/releasetools/build_image.py
build/tools/releasetools/common.py

device/oemMa/project/partition_size.mk
device/mediatek/build/build/tools/ptgen/MT6761/partition_table_MT6761_emmc_ab.csv ①相关配置项的注意 ,Type ,Download_File ,Download ,OTA_Update,FastBoot_Download, FastBoot_Erase ,FastBoot_Erase
device/mediatek/common/device.mk
device/mediatek/mt6761/init.mt6761.rc
device/mediatek/sepolicy/basic/non_plat/device.te
device/mediatek/sepolicy/basic/non_plat/non_plat/file.te
device/mediatek/sepolicy/basic/non_plat/non_plat/file_contexts // ⑧编译出现了failed with exit 4,  未知原因 "/mnt/vendor/secos" 还是 "/secos"
device/mediatek/sepolicy/basic/non_plat/non_plat/fsck.te
device/mediatek/sepolicy/basic/non_plat/non_plat/init.te
vendor/mediatek/proprietary/hardware/fstab/mt6765/fstab.in.mt6761   //⑤挂载参数错误导致的无法开机 ⑧"/mnt/vendor/secos" 还是 "/secos"； ⑥修改参数之后开机没有挂载上去skipping, slotselect参数; ⑦无法挂载在根目录/
vendor/mediatek/proprietary/scripts/releasetools/mt_ota_preprocess.py
vendor/mediatek/proprietary/scripts/releasetools/replace_img_from_target_files.py
```

### 修改分区表
修改的分区表的内容, 其中分区的大小的单位是 KB,
```
persist,EXT4,49152.0,,EMMC_USER,N,N,NONE,N,N,N,,N,,PROTECTED
antcommon,EXT4,102400,,EMMC_USER,N,Y,antcommon.img,N,N,N,,N,,AUTO
secos,EXT4,45120.0,,EMMC_USER,N,Y,secos.img,N,N,Y,,Y,,AUTO
sec1,Raw data,2048.0,,EMMC_USER,N,N,NONE,N,N,N,,N,,AUTO
proinfo,Raw data,3072.0,,EMMC_USER,N,N,NONE,N,N,N,,N,,PROTECTED
```
下面是对各个字段的说明:
Type, 文件的分区格式 有Raw data 和 EXT4
Download_File, 刷机的时的文件的文件名称 ,比如system.img, 注意前面的文件和未前面的文件
Download, 是否通过刷机的方式 ,应该和 Download_File 保持一致
OTA_Update,是否通过OTA升级的方式升级该分区, 如果加入的不是AB分区 ,建议写 N
FastBoot_Download,  是否通过FastBoot的方式下载
EMMC  指的是内存的
MTK 的代码, 会根据 这个 CSV 文件, 生辰一个 platform_Android_Scatter.txt 文件, 里面会包含分区的各种信息,
比如是否可以升级,1 刷机的文件名称, 其实的物理地址, 分区的大小, 分区的格式, 都是根据这个 csv 文件得来的.

### init.rc 创建目录
需要修改的 init.rc 文件.
```
device/mediatek/mt6761/init.mt6761.rc
```
```

# Create CIP mount point
    mkdir /custom

    mkdir /mnt/cd-rom 0000 system system

    mkdir /mnt/vendor/secos 0777 root root

# change lk_env permission
    chown root system /proc/lk_env
    chmod 0660 /proc/lk_env
```

```
    chown radio system /proc/freqhopping/dumpregs
    chown radio system /proc/freqhopping/freqhopping_debug
    chown radio system /proc/freqhopping/status
    chown radio system /proc/freqhopping/userdef
    chown system system /mnt/vendor/antcommon
    chmod 0777 /mnt/vendor/antcommon
    restorecon_recursive /mnt/vendor/antcommon
	# 改变路径的权限和用户以及用户组
    chown root root /mnt/vendor/secos
    chmod 0777 /mnt/vendor/secos
    restorecon_recursive /mnt/vendor/secos

# change owner
    chown system system /proc/bootprof
    chmod 0664 /proc/bootprof
    chown root system /proc/mtprof/cputime
    chown root system /proc/mtprof/reboot_pid

```
在这里, 我一开始想把这个分区挂载根目录下面, 也就是 /secos. 但是尝试了很多次, 都没有成功, 

### 挂载分区的 fstab 文件
vendor/mediatek/proprietary/hardware/fstab/mt6765/fstab.in.mt6761
```
DEVPATH(protect1)   /mnt/vendor/protect_f   ext4   FS_FLAG_COMMIT   FSMGR_FLAG_FMT
DEVPATH(protect2)   /mnt/vendor/protect_s   ext4   FS_FLAG_COMMIT   FSMGR_FLAG_FMT
DEVPATH(nvdata)     /mnt/vendor/nvdata      ext4   FS_FLAG_DISCARD  FSMGR_FLAG_FMT
DEVPATH(nvcfg)      /mnt/vendor/nvcfg       ext4   FS_FLAG_COMMIT   FSMGR_FLAG_FMT
DEVPATH(antcommon)  /mnt/vendor/antcommon   ext4   FS_FLAG_COMMIT   FSMGR_FLAG_FMT

DEVPATH(secos)      /mnt/vendor/secos   __MTK_DATAIMG_FSTYPE   defaults   FSMGR_FLAG_SYSTEM

#ifdef __PERSIST_PARTITION_SUPPORT
DEVPATH(persist)    /mnt/vendor/persist     ext4   FS_FLAG_COMMIT  FSMGR_FLAG_FMT
#endif

```
DEVPATH 是一个变量 ,关于路径的变量
```
#define DEVPATH(_part) /dev/block/platform/BOOTDEV/by-name/_part
```
defaults 是一个默认的参数集
__MTK_DATAIMG_FSTYPE 在上面定义为 ext4 格式的
```
#define FSMGR_FLAG_FMT  wait,check,formattable
```
### 添加分区的 selinux 权限
需要在 device/mediatek/sepolicy/basic/non_plat 路径下添加相关的 selinux 权限。

device/mediatek/sepolicy/basic/non_plat/device.te
```
type secos_block_device, dev_type;
```

device/mediatek/sepolicy/basic/non_plat/file.te
```
type secos_file, file_type, data_file_type;
```

device/mediatek/sepolicy/basic/non_plat/file_contexts
```
#############################
# secos files
/secos(/.*)?		u:object_r:secos_file:s0
/dev/block/platform/mtk-msdc\.0/[0-9]+\.(msdc|MSDC)0/by-name/secos	u:object_r:secos_block_device:s0
```
在这里我在这里踩得坑 ,因为我一开始要挂在 /mnt/vendor/ 目录下面的。 因此 ,我在这里把
/secos(/.*)? 写成了 /mnt/vendor/secos(/.*)? 。因此老是出现了编译错误: failed with error code 4 的错误。
 一开始并没有意识到这个错误。 导致在这里花费了很长的时间。这个将在随后说。
device/mediatek/sepolicy/basic/non_plat/fsck.te
```
allow fsck secos_block_device:blk_file rw_file_perms;
```
device/mediatek/sepolicy/basic/non_plat/init.te
```
allow init secos_block_device:blk_file { write read };
allow init secos_block_device:blk_file relabelto;
```


### 添加分区的 IMAGE 文件

添加分区的大小信息到一个 txt 文件中. 创建image 文件的时候, 需要用的到.
``` makefile
$(if $(BOARD_OEMIMAGE_PARTITION_SIZE),$(hide) echo "oem_size=$(BOARD_OEMIMAGE_PARTITION_SIZE)" >> $(1))
$(if $(BOARD_OEMIMAGE_JOURNAL_SIZE),$(hide) echo "oem_journal_size=$(BOARD_OEMIMAGE_JOURNAL_SIZE)" >> $(1))
$(if $(BOARD_OEMIMAGE_EXTFS_INODE_COUNT),$(hide) echo "oem_extfs_inode_count=$(BOARD_OEMIMAGE_EXTFS_INODE_COUNT)" >> $(1))
$(if $(BOARD_OEMIMAGE_EXTFS_RSV_PCT),$(hide) echo "oem_extfs_rsv_pct=$(BOARD_OEMIMAGE_EXTFS_RSV_PCT)" >> $(1))
$(if $(BOARD_ANTCOMMONIMAGE_PARTITION_SIZE),$(hide) echo "antcommon_size=$(BOARD_ANTCOMMONIMAGE_PARTITION_SIZE)" >> $(1))
$(if $(BOARD_SECOSIMAGE_PARTITION_SIZE),$(hide) echo "secos_size=$(BOARD_SECOSIMAGE_PARTITION_SIZE)" >> $(1))
$(if $(INTERNAL_USERIMAGES_SPARSE_EXT_FLAG),$(hide) echo "extfs_sparse_flag=$(INTERNAL_USERIMAGES_SPARSE_EXT_FLAG)" >> $(1))
```
即使你设置了 BOARD_SECOSIMAGE_PARTITION_SIZE, 但是 BOARD_SECOSIMAGE_PARTITION_SIZE 不一定是你设置的的大小．

device/mediatek/build/build/tools/ptgen/MT6761/ptgen.pl 里会根据分区的大小 , 各自对齐一下. 所以他的大小会改变．
比如我这里在分区表里给的大小时 45120 KB, 但是实际上得到的是 52 MB. 这个你可以在编译完的生成目录的 ninja 文件里搜索到实际的大小.
``` perl
if ($part->{Type} eq "EXT4" || $part->{Type} eq "FAT")
{
	$temp = $part->{Size_KB} * 1024;
	my $raw_name = GetABPartBaseName($part->{Partition_Name});
	if ($part->{Partition_Name} ne "${raw_name}_b")   # do not print for _b partition, its redundant setting
	{
		printf $part_size_fh ("BOARD_%sIMAGE_PARTITION_SIZE:=%d\n",uc($raw_name),$temp);
		if (exists $mountPointMapList{$raw_name})
		{
			if ($mountPointMapList{$raw_name} ne "")
			{
				printf $part_size_fh ("MTK_BOARD_ROOT_EXTRA_SYMLINKS += /mnt/vendor/%s:%s\n", $mountPointMapList{$raw_name}, $mountPointMapList{$raw_name});
			}
		}
		else
		{
			printf $part_size_fh ("MTK_BOARD_ROOT_EXTRA_SYMLINKS += /mnt/vendor/%s:%s\n", $raw_name, $raw_name);
		}
	}
}
if ($ArgList{MTK_BOARD_AVB_ENABLE} eq "true")
{
	if ($part->{Partition_Name} =~ /^boot(_a)?$/i || $part->{Partition_Name} =~ /^recovery?$/i)
	{
		$temp = $part->{Size_KB} * 1024;
		my $raw_name = GetABPartBaseName($part->{Partition_Name});
		printf $part_size_fh ("BOARD_%sIMAGE_PARTITION_SIZE:=%d\n",uc($raw_name),$temp);
	}
	if ($part->{Partition_Name} =~ /^dtbo(1|_a)?$/i)
	{
		$temp = $part->{Size_KB} * 1024;
		my $raw_name = GetPartBaseName($part->{Partition_Name});
		printf $part_size_fh ("BOARD_%sIMG_PARTITION_SIZE:=%d\n",uc($raw_name),$temp);
	}
}


```
将这些添加的宏, 放到编译 ROM 包里面.
``` makefile
# 建立 target files 与 secos.img 文件
# Depending on the various images guarantees that the underlying
# directories are up-to-date.
$(BUILT_TARGET_FILES_PACKAGE): \
		$(INSTALLED_BOOTIMAGE_TARGET) \
		$(MTK_BOOTIMAGE_TARGET) \
		$(INSTALLED_RADIOIMAGE_TARGET) \
		$(INSTALLED_RECOVERYIMAGE_TARGET) \
		$(FULL_SYSTEMIMAGE_DEPS) \
		$(INSTALLED_USERDATAIMAGE_TARGET) \
		$(INSTALLED_CACHEIMAGE_TARGET) \
		$(INSTALLED_SECOSIMAGE_TARGET) \
		$(INSTALLED_VENDORIMAGE_TARGET) \
		$(INSTALLED_PRODUCTIMAGE_TARGET) \
		$(INSTALLED_VBMETAIMAGE_TARGET) \
		$(INSTALLED_DTBOIMAGE_TARGET) \
		$(INTERNAL_SYSTEMOTHERIMAGE_FILES) \
		$(INSTALLED_ANDROID_INFO_TXT_TARGET) \
		$(INSTALLED_KERNEL_TARGET) \
		$(INSTALLED_2NDBOOTLOADER_TARGET) \

```

``` makefile
# -----------------------------------------------------------------
# secos image , 编译生成相关的 image 文件
INTERNAL_SECOSIMAGE_FILES := \
	$(filter $(TARGET_OUT_SECOS)/%,$(ALL_DEFAULT_INSTALLED_MODULES))

secosimage_intermediates := \
	$(call intermediates-dir-for,PACKAGING,secos)
BUILT_SECOSIMAGE_TARGET := $(PRODUCT_OUT)/secos.img

define build-secosimage-target
	$(call pretty,"Target secos fs image: $(INSTALLED_SECOSIMAGE_TARGET)")
	@mkdir -p $(TARGET_OUT_SECOS)
	@mkdir -p $(secosimage_intermediates) && rm -rf $(secosimage_intermediates)/secos_image_info.txt
	$(call generate-userimage-prop-dictionary, $(secosimage_intermediates)/secos_image_info.txt, skip_fsck=true)
	$(hide) PATH=$(foreach p,$(INTERNAL_USERIMAGES_BINARY_PATHS),$(p):)$$PATH \
	./build/tools/releasetools/build_image.py \
	$(TARGET_OUT_SECOS) $(secosimage_intermediates)/secos_image_info.txt $(INSTALLED_SECOSIMAGE_TARGET) $(TARGET_OUT)
	$(hide) $(call assert-max-image-size,$(INSTALLED_SECOSIMAGE_TARGET),$(BOARD_SECOSIMAGE_PARTITION_SIZE))
endef

# We just build this directly to the install location.
INSTALLED_SECOSIMAGE_TARGET := $(BUILT_SECOSIMAGE_TARGET)
$(INSTALLED_SECOSIMAGE_TARGET): $(INTERNAL_USERIMAGES_DEPS) $(INTERNAL_SECOSIMAGE_FILES)
	$(build-secosimage-target)

.PHONY: secosimage-nodeps
secosimage-nodeps: | $(INTERNAL_USERIMAGES_DEPS)
	$(build-secosimage-target)


```

将生成的 image 文件 添加到target_file 文件里. 如果你不需要升级的话, 感觉不需要添加这个的. 
``` makefile
ifdef BOARD_PREBUILT_VENDORIMAGE
	$(hide) mkdir -p $(zip_root)/IMAGES
	$(hide) cp $(INSTALLED_VENDORIMAGE_TARGET) $(zip_root)/IMAGES/
endif
# 生成的 image 文件放置 target_file 的IMAGES路径下面
ifdef BOARD_SECOSIMAGE_FILE_SYSTEM_TYPE
	$(hide) mkdir -p $(zip_root)/IMAGES
	$(hide) cp $(INSTALLED_SECOSIMAGE_TARGET) $(zip_root)/IMAGES/
endif
ifdef BOARD_PREBUILT_PRODUCTIMAGE
	$(hide) mkdir -p $(zip_root)/IMAGES
	$(hide) cp $(INSTALLED_PRODUCTIMAGE_TARGET) $(zip_root)/IMAGES/
endif
```

``` makefile
# 编译整个刷机包的
# Build files and then package it into the rom formats
.PHONY: droidcore
droidcore: files \
	systemimage \
	$(INSTALLED_BOOTIMAGE_TARGET) \
	$(MTK_BOOTIMAGE_TARGET) \
	$(INSTALLED_RECOVERYIMAGE_TARGET) \
	$(INSTALLED_VBMETAIMAGE_TARGET) \
	$(INSTALLED_USERDATAIMAGE_TARGET) \
	$(INSTALLED_CACHEIMAGE_TARGET) \
	$(INSTALLED_ANTCOMMONIMAGE_TARGET) \
	$(INSTALLED_SECOSIMAGE_TARGET) \
	$(INSTALLED_BPTIMAGE_TARGET) \
	$(INSTALLED_VENDORIMAGE_TARGET) \
	$(INSTALLED_PRODUCTIMAGE_TARGET) \
	$(INSTALLED_SYSTEMOTHERIMAGE_TARGET) \
	$(INSTALLED_FILES_FILE) \
	$(INSTALLED_FILES_FILE_VENDOR) \
	$(INSTALLED_FILES_FILE_PRODUCT) \
	$(INSTALLED_FILES_FILE_SYSTEMOTHER) \
	soong_docs

```

``` makefile
.PHONY: cacheimage
cacheimage: $(INSTALLED_CACHEIMAGE_TARGET)

# 添加单独编译的代码
.PHONY: secosimage
secosimage: $(INSTALLED_SECOSIMAGE_TARGET)

.PHONY: bptimage
bptimage: $(INSTALLED_BPTIMAGE_TARGET)

```

build/tools/releasetools/build_image.py
``` py
  elif mount_point == "oem":
    copy_prop("fs_type", "fs_type")
    copy_prop("oem_size", "partition_size")
    if not copy_prop("oem_journal_size", "journal_size"):
      d["journal_size"] = "0"
    copy_prop("oem_extfs_inode_count", "extfs_inode_count")
    if not copy_prop("oem_extfs_rsv_pct", "extfs_rsv_pct"):
      d["extfs_rsv_pct"] = "0"
  elif mount_point == "secos":
    # out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt 读取的是这个文件
    copy_prop("fs_type", "fs_type")
    copy_prop("secos_size", "partition_size")
    copy_prop("secos_verity_block_device", "verity_block_device")
  d["partition_name"] = mount_point
  return d

```

``` py
    elif image_filename == "product.img":
      mount_point = "product"
    elif image_filename == "secos.img":
      mount_point = "secos"
    else:
      print("error: unknown image file name ", image_filename, file=sys.stderr)
      sys.exit(1)
```


## 出现的错误

### 错误索引

```
相关配置项的注意 ,Type ,Download_File ,Download ,OTA_Update ,FastBoot_Download, 
命令行之前需要有一个TAB,
生成了相关的image 但是没有放在 target_files_package
编译出现了failed with exit 4 , "/mnt/vendor/secos" 还是 "/secos"
挂载参数错误导致的无法开机
修改参数之后开机没有挂载上去 skipping, slotselect参数
无法挂载在根目录/
```

### 分区表的各项注意
Type ,  文件的分区格式 有Raw data 和 EXT4
Download_File , 刷机的时的文件的文件名称 ,比如system.img
Download ,    是否通过刷机的方式 ,应该和 Download_File保持一致
OTA_Update ,   是否通过OTA升级的方式升级该分区 ,如果加入的不是AB分区 ,建议写N
FastBoot_Download,  是否通过FastBoot的方式下载


### missing separator 错误
```
mobicore - including vendor/mediatek/proprietary/trustzone/trustonic/source/external/mobicore/common/400b/Android.mk
including vendor/mediatek/widevine_tlib/Android.mk ...
[950/950] including vendor/nxp/Android.mk ...
build/make/core/Makefile:2920: error: missing separator.
14:30:18 ckati failed with: exit status 1
***************************************************
  end android build
***************************************************
```
这个是因为Makefile 里使用命令行之前 ,需要空出TAB键

### failed with exit code 4
```
img-1 size:0x5c0
add index1 image, this is final image
img name not match,exit!!
[  0% 27/50746] build check-kernel-config
[  0% 28/50746] Copying: out/target/product/ax2129/system/etc/tee.img
[  0% 29/50746] Target secos fs image: out/target/product/ax2129/secos.img
FAILED: out/target/product/ax2129/secos.img 
/bin/bash -c "(mkdir -p out/target/product/ax2129/secos ) && (mkdir -p out/target/product/ax2129/obj/PACKAGING/secos_intermediates && rm -rf out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"ext_mkuserimg=mkuserimg_mke2fs.sh\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"fs_type=ext4\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"system_size=3221225472\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"userdata_size=2778464256\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"vendor_fs_type=ext4\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"vendor_size=838860800\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"extfs_sparse_flag=-s\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"squashfs_sparse_flag=-s\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"selinux_fc=out/target/product/ax2129/obj/ETC/file_contexts.bin_intermediates/file_contexts.bin\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"system_verity_block_device=/dev/block/platform/bootdevice/by-name/system\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"vendor_verity_block_device=/dev/block/platform/bootdevice/by-name/vendor\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_avbtool=avbtool\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_system_hashtree_enable=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_system_add_hashtree_footer_args=--rollback_index 0 --setup_as_rootfs_from_kernel\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_system_key_path=device/mediatek/common/system_prvk.pem\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_system_algorithm=SHA256_RSA2048\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_system_rollback_index_location=2\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_vendor_hashtree_enable=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_vendor_add_hashtree_footer_args=\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_product_hashtree_enable=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_product_add_hashtree_footer_args=\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"recovery_as_boot=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"system_root_image=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt; echo \"ramdisk_dir=out/target/product/ax2129/root\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"skip_fsck=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"secos_size=50331648\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"secos_fs_type=ext4\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (PATH=out/host/linux-x86/bin/:\$PATH ./build/tools/releasetools/build_image.py out/target/product/ax2129/secos out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt out/target/product/ax2129/secos.img out/target/product/ax2129/system ) && (size=\$(for i in out/target/product/ax2129/secos.img; do stat --format \"%s\" \"\$i\" | tr -d '\\n'; echo +; done; echo 0); total=\$(( \$( echo \"\$size\" ) )); printname=\$(echo -n \"out/target/product/ax2129/secos.img\" | tr \" \" +); maxsize=\$((50331648)); if [ \"\$total\" -gt \"\$maxsize\" ]; then echo \"error: \$printname too large (\$total > \$maxsize)\"; false; elif [ \"\$total\" -gt \$((maxsize - 32768)) ]; then echo \"WARNING: \$printname approaching size limit (\$total now; limit \$maxsize)\"; fi )"
error: failed to build out/target/product/ax2129/secos.img from out/target/product/ax2129/secos
Error: '['mkuserimg_mke2fs.sh', '-s', 'out/target/product/ax2129/secos', 'out/target/product/ax2129/secos.img', 'ext4', 'secos', '50331648', '-D', 'out/target/product/ax2129/system', '-L', 'secos', 'out/target/product/ax2129/obj/ETC/file_contexts.bin_intermediates/file_contexts.bin']' failed with exit code 4:
MKE2FS_CONFIG=./system/extras/ext4_utils/mke2fs.conf mke2fs -L secos -E android_sparse -t ext4 -b 4096 out/target/product/ax2129/secos.img 12288
mke2fs 1.43.3 (04-Sep-2016)
Creating filesystem with 12288 4k blocks and 12288 inodes

Allocating group tables: 0/1   done                            
Writing inode tables: 0/1   done                            
Creating journal (1024 blocks): done
Writing superblocks and filesystem accounting information: 0/1   done

e2fsdroid -p out/target/product/ax2129/system -S out/target/product/ax2129/obj/ETC/file_contexts.bin_intermediates/file_contexts.bin -f out/target/product/ax2129/secos -a /secos out/target/product/ax2129/secos.img
set_selinux_xattr: Unknown code ____ 255 searching for label "/secos"

[  0% 30/50746] sign: out/target/product/ax2129/tee.img
writing RSA key
writing RSA key
writing RSA key

```
一开始我以为时 mkuserimg_mke2fs.sh 这个行的命令行出现了错误 ,但是把这个命令行单独拿出来 ,执行命令 ,没有发现任何错误．
后来我怀疑是后面的的计算image 大小的出现了错误． 这个image的大小超出了系统分配的分区的大小．实际上也不是真正的错误原因.
```
(size=\$(for i in out/target/product/ax2129/secos.img; do stat --format \"%s\" \"\$i\" | tr -d '\\n'; echo +; done; echo 0); total=\$(( \$( echo \"\$size\" ) )); printname=\$(echo -n \"out/target/product/ax2129/secos.img\" | tr \" \" +); maxsize=\$((50331648)); if [ \"\$total\" -gt \"\$maxsize\" ]; then echo \"error: \$printname too large (\$total > \$maxsize)\"; false; elif [ \"\$total\" -gt \$((maxsize - 32768)) ]; then echo \"WARNING: \$printname approaching size limit (\$total now; limit \$maxsize)\"; fi )"
```
out/host/linux-x86/bin/mkuserimg_mke2fs.sh
system/extras/ext4_utils/mkuserimg_mke2fs.sh

``` makefile
# $(1): The file to check
define get-file-size
stat --format "%s" "$(1)" | tr -d '\n'
endef

```

``` makefile
# INSTALLED_SECOSIMAGE_TARGET  out/target/product/ax2129/secos.img 
# BOARD_SECOSIMAGE_PARTITION_SIZE  
# $(hide) $(call assert-max-image-size,$(INSTALLED_SECOSIMAGE_TARGET),$(BOARD_SECOSIMAGE_PARTITION_SIZE))
# $(1): The file(s) to check (often $@)
# $(2): The partition size.
define assert-max-image-size
$(if $(2), \
  size=$$(for i in $(1); do $(call get-file-size,$$i); echo +; done; echo 0); \
  total=$$(( $$( echo "$$size" ) )); \
  printname=$$(echo -n "$(1)" | tr " " +); \
  maxsize=$$(($(2))); \
  if [ "$$total" -gt "$$maxsize" ]; then \
    echo "error: $$printname too large ($$total > $$maxsize)"; \
    false; \
  elif [ "$$total" -gt $$((maxsize - 32768)) ]; then \
    echo "WARNING: $$printname approaching size limit ($$total now; limit $$maxsize)"; \
  fi \
 , \
  true \
 )
endef
```
但是经过详细的计算 ,得出还是没超过．所以这个也不似root cause．

随后 ,请教别人, 出现错误的地方是后面的 selinux相关的．
```
e2fsdroid -p out/target/product/ax2129/system -S out/target/product/ax2129/obj/ETC/file_contexts.bin_intermediates/file_contexts.bin -f out/target/product/ax2129/secos -a /secos out/target/product/ax2129/secos.img
set_selinux_xattr: Unknown code ____ 255 searching for label "/secos"
```
e2fsdroid 是一个命令。他的源码在 external/e2fsprogs 下面。
``` cpp
// external/e2fsprogs/contrib/android/perms.c
static errcode_t set_selinux_xattr(ext2_filsys fs, ext2_ino_t ino,
				   struct inode_params *params)
{
	errcode_t retval;
	char *secontext = NULL;
	struct ext2_inode inode;

	if (params->sehnd == NULL)
		return 0;

	retval = ext2fs_read_inode(fs, ino, &inode);
	if (retval) {
		com_err(__func__, retval,
			_("while reading inode %u"), ino);
		return retval;
	}

	retval = selabel_lookup(params->sehnd, &secontext, params->filename,
				inode.i_mode);
	if (retval < 0) {
		com_err(__func__, retval,
			_("searching for label \"%s\""), params->filename);
		exit(1);
	}

	retval = ino_add_xattr(fs, ino,  "security." XATTR_SELINUX_SUFFIX,
			       secontext, strlen(secontext) + 1);

	freecon(secontext);
	return retval;
}

```
有此可以知道他进入了 selabel_lookup 函数
``` cpp
// external/selinux/libselinux/src/label.c
int selabel_lookup(struct selabel_handle *rec, char **con,
		   const char *key, int type)
{
	struct selabel_lookup_rec *lr;

	lr = selabel_lookup_common(rec, 1, key, type);
	if (!lr)
		return -1;

	*con = strdup(lr->ctx_trans);
	return *con ? 0 : -1;
}
```
这个错误的主要原因是 /secos(/.*)? 写成了 /mnt/vendor/secos(/.*)? 还不太理解为什么不能这么写
```
sig:out/target/product/ax2129/sig/tee.sig
output path:out/target/product/ax2129/tee-verified.img
[  0% 46/50745] Export includes file: out/target/product/ax2129/obj/STATIC_LIBRARIES/libcam.halsensor.custom_intermediates/custgen.config_metadata.h -- out/target/product/ax2129/obj_arm/STATIC_LIBRARIES/libcam.halsensor.custom_intermediates/export_includes
[  0% 47/50744] Export includes file: out/target/product/ax2129/obj/STATIC_LIBRARIES/libcam.halsensor.custom_intermediates/custgen.config_metadata.h -- out/target/product/ax2129/obj/STATIC_LIBRARIES/libcam.halsensor.custom_intermediates/export_includes
[  0% 48/50743] //prebuilts/sdk/current/extras/material-design:android-support-design-bottomsheet-nodeps overlay resource file list [common]
[  0% 49/50743] Target secos fs image: out/target/product/ax2129/secos.img
FAILED: out/target/product/ax2129/secos.img 
/bin/bash -c "(mkdir -p out/target/product/ax2129/secos ) && (mkdir -p out/target/product/ax2129/obj/PACKAGING/secos_intermediates && rm -rf out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"ext_mkuserimg=mkuserimg_mke2fs.sh\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"fs_type=ext4\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"system_size=3221225472\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"userdata_size=2778464256\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"vendor_fs_type=ext4\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"vendor_size=838860800\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"extfs_sparse_flag=-s\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"squashfs_sparse_flag=-s\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"selinux_fc=out/target/product/ax2129/obj/ETC/file_contexts.bin_intermediates/file_contexts.bin\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"system_verity_block_device=/dev/block/platform/bootdevice/by-name/system\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"vendor_verity_block_device=/dev/block/platform/bootdevice/by-name/vendor\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_avbtool=avbtool\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_system_hashtree_enable=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_system_add_hashtree_footer_args=--rollback_index 0 --setup_as_rootfs_from_kernel\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_system_key_path=device/mediatek/common/system_prvk.pem\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_system_algorithm=SHA256_RSA2048\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_system_rollback_index_location=2\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_vendor_hashtree_enable=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_vendor_add_hashtree_footer_args=\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_product_hashtree_enable=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"avb_product_add_hashtree_footer_args=\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"recovery_as_boot=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"system_root_image=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt; echo \"ramdisk_dir=out/target/product/ax2129/root\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"skip_fsck=true\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"secos_size=50331648\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (echo \"secos_fs_type=ext4\" >>  out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt ) && (PATH=out/host/linux-x86/bin/:\$PATH ./build/tools/releasetools/build_image.py out/target/product/ax2129/secos out/target/product/ax2129/obj/PACKAGING/secos_intermediates/secos_image_info.txt out/target/product/ax2129/secos.img ) && (size=\$(for i in out/target/product/ax2129/secos.img; do stat --format \"%s\" \"\$i\" | tr -d '\\n'; echo +; done; echo 0); total=\$(( \$( echo \"\$size\" ) )); printname=\$(echo -n \"out/target/product/ax2129/secos.img\" | tr \" \" +); maxsize=\$((50331648)); if [ \"\$total\" -gt \"\$maxsize\" ]; then echo \"error: \$printname too large (\$total > \$maxsize)\"; false; elif [ \"\$total\" -gt \$((maxsize - 32768)) ]; then echo \"WARNING: \$printname approaching size limit (\$total now; limit \$maxsize)\"; fi )"

Builds output_image from the given input_directory, properties_file,
and writes the image to target_output_directory.

Usage:  build_image.py input_directory properties_file output_image \
            target_output_directory

[  0% 50/50743] //prebuilts/sdk/current/extras/material-design:android-support-design-bottomsheet-nodeps aapt2 link [common]
[  0% 51/50743] build /mnt/code/huaqin/AX2129_D/out/target/product/ax2129/obj/BOOTLOADER_OBJ/build-ax2129/lk.img
make: Entering directory `/mnt/code/huaqin/AX2129_D/vendor/mediatek/proprietary/bootable/bootloader/lk'


```

### 挂载的时候跳过了, 缺少 slotselect 参数

添加一个AB分区的时候
未添加 slotselect 参数导致的无法挂载, Skipping

```
[   23.075795] .(3)[273:wdtk-3][wdk-c] cpu=3,lbit=0x9,cbit=0xf,0,1,2930432545,[23075790516,19999990,43072185]
[   23.075804] .(3)[273:wdtk-3][thread:273][RT:23075799901] 2019-09-05 12:52:45.905161 UTC;android time 2019-09-05 12:52:45.905161
[   23.084361] .(2)[272:wdtk-2][wdk-c] cpu=2,lbit=0xd,cbit=0xf,0,1,2930432545,[23084350593,19991430,43072185]
[   23.140353] .(1)[271:wdtk-1][wdk-k] cpu=1,lbit=0xf,cbit=0xf,0,1,2930432545,[23140341209,19935439,43072185]
[   24.294772] .(1)[303:init]init: [libfs_mgr]Skipping '/dev/block/platform/bootdevice/by-name/secos' during mount_all
[   24.299752] .(2)[303:init]init: [libfs_mgr]superblock s_max_mnt_count:65535,/dev/block/platform/bootdevice/by-name/persist
[   24.301783] .(2)[303:init]EXT4-fs (mmcblk0p11): Ignoring removed nomblk_io_submit option
[   24.304515] .(2)[303:init]EXT4-fs (mmcblk0p11): mounted filesystem with ordered data mode. Opts: errors=remount-ro,nomblk_io_submit
[   24.306406] .(2)[303:init]init: [libfs_mgr]check_fs(): mount(/dev/block/platform/bootdevice/by-name/persist,/mnt/vendor/persist,ext4)=0: Success
[   24.309307] .(2)[303:init]init: [libfs_mgr]check_fs(): unmount(/mnt/vendor/persist) succeeded

```
以下时挂载的代码.
```cpp
// system/core/fs_mgr/fs_mgr.cpp
int fs_mgr_mount_all(struct fstab *fstab, int mount_mode)
{
    int i = 0;
    int encryptable = FS_MGR_MNTALL_DEV_NOT_ENCRYPTABLE;
    int error_count = 0;
    int mret = -1;
    int mount_errno = 0;
    int attempted_idx = -1;
    FsManagerAvbUniquePtr avb_handle(nullptr);

    if (!fstab) {
        return FS_MGR_MNTALL_FAIL;
    }

    for (i = 0; i < fstab->num_entries; i++) {
        /* Don't mount entries that are managed by vold or not for the mount mode*/
        if ((fstab->recs[i].fs_mgr_flags & (MF_VOLDMANAGED | MF_RECOVERYONLY)) ||
             ((mount_mode == MOUNT_MODE_LATE) && !fs_mgr_is_latemount(&fstab->recs[i])) ||
             ((mount_mode == MOUNT_MODE_EARLY) && fs_mgr_is_latemount(&fstab->recs[i]))) {
            continue;
        }

        /* Skip swap and raw partition entries such as boot, recovery, etc */
        if (!strcmp(fstab->recs[i].fs_type, "swap") ||
            !strcmp(fstab->recs[i].fs_type, "emmc") ||
            !strcmp(fstab->recs[i].fs_type, "mtd")) {
            continue;
        }

        /* Skip mounting the root partition, as it will already have been mounted */
        if (!strcmp(fstab->recs[i].mount_point, "/")) {
            if ((fstab->recs[i].fs_mgr_flags & MS_RDONLY) != 0) {
                fs_mgr_set_blk_ro(fstab->recs[i].blk_device);
            }
            continue;
        }

        /* Translate LABEL= file system labels into block devices */
        if (is_extfs(fstab->recs[i].fs_type)) {
            int tret = translate_ext_labels(&fstab->recs[i]);
            if (tret < 0) {
                LERROR << "Could not translate label to block device";
                continue;
            }
        }

        if (fstab->recs[i].fs_mgr_flags & MF_WAIT &&
            !fs_mgr_wait_for_file(fstab->recs[i].blk_device, 20s)) {
            LERROR << "Skipping '" << fstab->recs[i].blk_device << "' during mount_all";
            continue;
        }

        if (fstab->recs[i].fs_mgr_flags & MF_AVB) {
            if (!avb_handle) {
                avb_handle = FsManagerAvbHandle::Open(*fstab);
                if (!avb_handle) {
                    LERROR << "Failed to open FsManagerAvbHandle";
                    return FS_MGR_MNTALL_FAIL;
                }
            }
            if (avb_handle->SetUpAvbHashtree(&fstab->recs[i], true /* wait_for_verity_dev */) ==
                SetUpAvbHashtreeResult::kFail) {
                LERROR << "Failed to set up AVB on partition: "
                       << fstab->recs[i].mount_point << ", skipping!";
                /* Skips mounting the device. */
                continue;
            }
        } else if ((fstab->recs[i].fs_mgr_flags & MF_VERIFY)) {
            int rc = fs_mgr_setup_verity(&fstab->recs[i], true);
            if (__android_log_is_debuggable() &&
                    (rc == FS_MGR_SETUP_VERITY_DISABLED ||
                     rc == FS_MGR_SETUP_VERITY_SKIPPED)) {
                LINFO << "Verity disabled";
            } else if (rc != FS_MGR_SETUP_VERITY_SUCCESS) {
                LERROR << "Could not set up verified partition, skipping!";
                continue;
            }
        }

        int last_idx_inspected;
        int top_idx = i;

        mret = mount_with_alternatives(fstab, i, &last_idx_inspected, &attempted_idx);
        i = last_idx_inspected;
        mount_errno = errno;

        /* Deal with encryptability. */
        if (!mret) {
            int status = handle_encryptable(&fstab->recs[attempted_idx]);

            if (status == FS_MGR_MNTALL_FAIL) {
                /* Fatal error - no point continuing */
                return status;
            }

            if (status != FS_MGR_MNTALL_DEV_NOT_ENCRYPTABLE) {
                if (encryptable != FS_MGR_MNTALL_DEV_NOT_ENCRYPTABLE) {
                    // Log and continue
                    LERROR << "Only one encryptable/encrypted partition supported";
                }
                encryptable = status;
                if (status == FS_MGR_MNTALL_DEV_NEEDS_METADATA_ENCRYPTION) {
                    if (!call_vdc(
                            {"cryptfs", "encryptFstab", fstab->recs[attempted_idx].mount_point})) {
                        LERROR << "Encryption failed";
                        return FS_MGR_MNTALL_FAIL;
                    }
                }
            }

            /* Success!  Go get the next one */
            continue;
        }

        bool wiped = partition_wiped(fstab->recs[top_idx].blk_device);
        bool crypt_footer = false;
        if (mret && mount_errno != EBUSY && mount_errno != EACCES &&
            fs_mgr_is_formattable(&fstab->recs[top_idx]) && wiped) {
            /* top_idx and attempted_idx point at the same partition, but sometimes
             * at two different lines in the fstab.  Use the top one for formatting
             * as that is the preferred one.
             */
            LERROR << __FUNCTION__ << "(): " << fstab->recs[top_idx].blk_device
                   << " is wiped and " << fstab->recs[top_idx].mount_point
                   << " " << fstab->recs[top_idx].fs_type
                   << " is formattable. Format it.";
            if (fs_mgr_is_encryptable(&fstab->recs[top_idx]) &&
                strcmp(fstab->recs[top_idx].key_loc, KEY_IN_FOOTER)) {
                int fd = open(fstab->recs[top_idx].key_loc, O_WRONLY);
                if (fd >= 0) {
                    LINFO << __FUNCTION__ << "(): also wipe "
                          << fstab->recs[top_idx].key_loc;
                    wipe_block_device(fd, get_file_size(fd));
                    close(fd);
                } else {
                    PERROR << __FUNCTION__ << "(): "
                           << fstab->recs[top_idx].key_loc << " wouldn't open";
                }
            } else if (fs_mgr_is_encryptable(&fstab->recs[top_idx]) &&
                !strcmp(fstab->recs[top_idx].key_loc, KEY_IN_FOOTER)) {
                crypt_footer = true;
            }
            if (fs_mgr_do_format(&fstab->recs[top_idx], crypt_footer) == 0) {
                /* Let's replay the mount actions. */
                i = top_idx - 1;
                continue;
            } else {
                LERROR << __FUNCTION__ << "(): Format failed. "
                       << "Suggest recovery...";
                encryptable = FS_MGR_MNTALL_DEV_NEEDS_RECOVERY;
                continue;
            }
        }

        /* mount(2) returned an error, handle the encryptable/formattable case */
        if (mret && mount_errno != EBUSY && mount_errno != EACCES &&
            fs_mgr_is_encryptable(&fstab->recs[attempted_idx])) {
            if (wiped) {
                LERROR << __FUNCTION__ << "(): "
                       << fstab->recs[attempted_idx].blk_device
                       << " is wiped and "
                       << fstab->recs[attempted_idx].mount_point << " "
                       << fstab->recs[attempted_idx].fs_type
                       << " is encryptable. Suggest recovery...";
                encryptable = FS_MGR_MNTALL_DEV_NEEDS_RECOVERY;
                continue;
            } else {
                /* Need to mount a tmpfs at this mountpoint for now, and set
                 * properties that vold will query later for decrypting
                 */
                LERROR << __FUNCTION__ << "(): possibly an encryptable blkdev "
                       << fstab->recs[attempted_idx].blk_device
                       << " for mount " << fstab->recs[attempted_idx].mount_point
                       << " type " << fstab->recs[attempted_idx].fs_type;
                if (fs_mgr_do_tmpfs_mount(fstab->recs[attempted_idx].mount_point) < 0) {
                    ++error_count;
                    continue;
                }
            }
            encryptable = FS_MGR_MNTALL_DEV_MIGHT_BE_ENCRYPTED;
        } else if (mret && mount_errno != EBUSY && mount_errno != EACCES &&
                   should_use_metadata_encryption(&fstab->recs[attempted_idx])) {
            if (!call_vdc({"cryptfs", "mountFstab", fstab->recs[attempted_idx].mount_point})) {
                ++error_count;
            }
            encryptable = FS_MGR_MNTALL_DEV_IS_METADATA_ENCRYPTED;
            continue;
        } else {
            // fs_options might be null so we cannot use PERROR << directly.
            // Use StringPrintf to output "(null)" instead.
            if (fs_mgr_is_nofail(&fstab->recs[attempted_idx])) {
                PERROR << android::base::StringPrintf(
                    "Ignoring failure to mount an un-encryptable or wiped "
                    "partition on %s at %s options: %s",
                    fstab->recs[attempted_idx].blk_device, fstab->recs[attempted_idx].mount_point,
                    fstab->recs[attempted_idx].fs_options);
            } else {
                PERROR << android::base::StringPrintf(
                    "Failed to mount an un-encryptable or wiped partition "
                    "on %s at %s options: %s",
                    fstab->recs[attempted_idx].blk_device, fstab->recs[attempted_idx].mount_point,
                    fstab->recs[attempted_idx].fs_options);
                ++error_count;
            }
            continue;
        }
    }

    if (error_count) {
        return FS_MGR_MNTALL_FAIL;
    } else {
        return encryptable;
    }
}
```


### 无法开机, 启动后只进入到 bootlogo
开机只到bootlogo 阶段 , 出现了 hwcomposer 错误 , 导致无法开机 , 这个是因为挂载参数的错误

```
01-01 00:23:45.730   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #13 checkModemReady fail...
01-01 00:23:45.810   435   435 I ccci_mdinit: (1):check_nvram_ready(), property_get("vendor.service.nvram_init") = , read_nvram_ready_retry = 5
01-01 00:23:45.816   458   458 W stp_dump: fail to get Sdcard stat :No such file or directory, count: 2
01-01 00:23:45.830   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #14 checkModemReady fail...
01-01 00:23:45.927   379   472 D AudioCustParam: checkNvramReady(), property_get("vendor.service.nvram_init") = , read_nvram_ready_retry = 4
01-01 00:23:45.930   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #15 checkModemReady fail...
01-01 00:23:46.030   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #16 checkModemReady fail...
01-01 00:23:46.067   456   456 I ADB_SERVICES: service_to_fd shell,v2,TERM=xterm:export ANDROID_LOG_TAGS="''"; exec logcat
01-01 00:23:46.099   456   456 I ADB_SERVICES: local_socket_flush_outgoing read_data=3958
01-01 00:23:46.130   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #17 checkModemReady fail...
01-01 00:23:46.138   359   408 D TeeRegistryUpdate: Wait a bit longer for registry to appear [vendor/mediatek/proprietary/trustzone/trustonic/source/external/mobicore/common/400b/Daemon/src/RegistryUpdate.cpp:89]
01-01 00:23:46.206   388   388 E hwcomposer: [HWC] Can't get PQ service tried(21) times  
01-01 00:23:46.231   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #18 checkModemReady fail...
01-01 00:23:46.307   388   388 E hwcomposer: [HWC] Can't get PQ service tried(22) times  
01-01 00:23:46.311   435   435 I ccci_mdinit: (1):check_nvram_ready(), property_get("vendor.service.nvram_init") = , read_nvram_ready_retry = 6
01-01 00:23:46.331   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #19 checkModemReady fail...
01-01 00:23:46.408   388   388 E hwcomposer: [HWC] Can't get PQ service tried(23) times  
01-01 00:23:46.427   379   472 D AudioCustParam: checkNvramReady(), property_get("vendor.service.nvram_init") = , read_nvram_ready_retry = 5
01-01 00:23:46.431   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #20 checkModemReady fail...
01-01 00:23:46.509   388   388 E hwcomposer: [HWC] Can't get PQ service tried(24) times  
01-01 00:23:46.531   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #21 checkModemReady fail...
01-01 00:23:46.599   417   417 I wmt_launcher: (persist.vendor.connsys.fwlog.status) is not supported
01-01 00:23:46.610   388   388 E hwcomposer: [HWC] Can't get PQ service tried(25) times  
01-01 00:23:46.631   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #22 checkModemReady fail...
01-01 00:23:46.711   388   388 E hwcomposer: [HWC] Can't get PQ service tried(26) times  
01-01 00:23:46.731   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #23 checkModemReady fail...
01-01 00:23:46.811   435   435 I ccci_mdinit: (1):check_nvram_ready(), property_get("vendor.service.nvram_init") = , read_nvram_ready_retry = 7
01-01 00:23:46.812   388   388 E hwcomposer: [HWC] Can't get PQ service tried(27) times  
01-01 00:23:46.831   379   496 W SpeechMessengerNormal: formatShareMemoryThread(), #24 checkModemReady fail...
01-01 00:23:46.913   388   388 E hwcomposer: [HWC] Can't get PQ service tried(28) times  


```
一般情况是这个是第四列或者第五列的挂载参数有问题。
```
#define DEVPATH(_part) /dev/block/platform/BOOTDEV/by-name/_part
DEVPATH(secos)      /mnt/vendor/secos   __MTK_DATAIMG_FSTYPE   defaults   FSMGR_FLAG_SYSTEM
```
我是第五列参数设置的不对, 以下是我最开始的错误的挂载配置．　修改为上面的就正常了.
```
#define DEVPATH(_part) /dev/block/platform/BOOTDEV/by-name/_part
DEVPATH(secos)      /mnt/vendor/secos   __MTK_DATAIMG_FSTYPE   defaults   defaults
```

###　无法找到 secos.img
```

vbmeta.img already exists, no need to overwrite...
boot.img already exists, no need to overwrite...
system.img already exists, no need to overwrite...
lk.img already exists, no need to overwrite...
preloader.img already exists, no need to overwrite...
Traceback (most recent call last):
  File "build/make/tools/releasetools/add_img_to_target_files", line 885, in <module>
    main(sys.argv[1:])
  File "build/make/tools/releasetools/add_img_to_target_files", line 879, in main
    AddImagesToTargetFiles(args[0])
  File "build/make/tools/releasetools/add_img_to_target_files", line 830, in AddImagesToTargetFiles
    AddRadioImagesForAbOta(output_zip, ab_partitions)
  File "build/make/tools/releasetools/add_img_to_target_files", line 570, in AddRadioImagesForAbOta
    assert os.path.exists(img_path), "cannot find " + img_name
AssertionError: cannot find secos.img
ninja: build stopped: subcommand failed.
14:51:27 ninja failed with: exit status 1
```

这个表明你需要把生成的 secos.img 文件放到 IMAGES 目录下面. 可以通过上面的 Makefile 做到.

### 无法挂载在根目录
系统启动的时候 ,找不到 /secos 目录

```
01-01 00:00:57.039   379  2678 D vold    : Starting trim of /mnt/vendor/protect_f
01-01 00:00:57.039   379  2678 W vold    : Failed to open /mnt/vendor/protect_f: Permission denied
01-01 00:00:57.039   379  2678 D vold    : Starting trim of /mnt/vendor/protect_s
01-01 00:00:57.039   379  2678 W vold    : Failed to open /mnt/vendor/protect_s: Permission denied
01-01 00:00:57.040   379  2678 D vold    : Starting trim of /mnt/vendor/nvdata
01-01 00:00:57.040   379  2678 W vold    : Failed to open /mnt/vendor/nvdata: Permission denied
01-01 00:00:57.040   379  2678 D vold    : Starting trim of /mnt/vendor/nvcfg
01-01 00:00:57.040   379  2678 W vold    : Failed to open /mnt/vendor/nvcfg: Permission denied
01-01 00:00:57.040   379  2678 D vold    : Starting trim of /secos
01-01 00:00:57.040   379  2678 W vold    : Failed to open /secos: No such file or directory
01-01 00:00:57.040   379  2678 D vold    : Starting trim of /mnt/vendor/persist
01-01 00:00:57.040   379  2678 W vold    : Failed to open /mnt/vendor/persist: Permission denied
01-01 00:00:57.045   429   553 D MDP     : DpAsyncBlitStream: rot(0) flip(0) padding(c)
```
这个是因为 , 系统启动的时候并没有创建 /secos 目录 , 无法打开 secos 目录。
不知道为什么总是无法创建根目录下的secos目录。即使我在 mkdir /system 的后面, 创建 /secos 路径, 也不行. ???

### OTA 再次验证
如果你添加了一个分区 , 一定要再次看看 OTA升级功能是否正常
```
09-09 09:14:08.198  1056  2680 W BestClock: java.time.DateTimeException: Missing NTP fix
09-09 09:14:08.201  1056  2681 W BestClock: java.time.DateTimeException: Missing NTP fix
09-09 09:14:08.205   377   377 I bootctrlHAL: boot control bootctrl_get_suffix
09-09 09:14:08.206   614   614 E update_engine: [0909/091408.206236:ERROR:boot_control_android.cc(123)] Device file /dev/block/platform/bootdevice/by-name/antcommon_a does not exist.
09-09 09:14:08.208   377   377 I bootctrlHAL: boot control bootctrl_get_suffix
09-09 09:14:08.208   614   614 E update_engine: [0909/091408.208489:ERROR:boot_control_android.cc(123)] Device file /dev/block/platform/bootdevice/by-name/antcommon_b does not exist.
09-09 09:14:08.210   377   377 I bootctrlHAL: boot control bootctrl_get_suffix
09-09 09:14:08.244   377   377 I chatty  : uid=0(root) boot@1.0-servic identical 21 lines
09-09 09:14:08.246   377   377 I bootctrlHAL: boot control bootctrl_get_suffix
09-09 09:14:08.247  1056  2680 W BestClock: java.time.DateTimeException: Missing NTP fix
09-09 09:14:08.248   377   377 I bootctrlHAL: boot control bootctrl_get_suffix
09-09 09:14:08.248   614   614 E update_engine: [0909/091408.248628:ERROR:delta_performer.cc(829)] Unable to determine all the partition devices.
09-09 09:14:08.248   614   614 E update_engine: [0909/091408.248856:ERROR:download_action.cc(337)] Error ErrorCode::kInstallDeviceOpenError (7) in DeltaPerformer's Write method when processing the received payload -- Terminating processing
09-09 09:14:08.250  3919  3919 I ADB_SERVICES: for fd 47, revents = 2019
09-09 09:14:08.250  3919  3919 I ADB_SERVICES: for fd 47, revents = 2019
```
在编译生成的full_project-target_files(obj/PACKAGING/target_files_intermediates/full_project-target_files/META/ab_partitions.txt)
或者该 full_project-target_files.zip 的目录下找到 META 目录 ,该目录下的 ab_partitions.txt 
文件保存着系统可以升级的分区, 如果你添加的分区时AB分区 ,所以时可以升级的, 在分区表的csv文件的 OTA_Update 字段是 Y.
以下是 ab_partitions.txt 文件的内容
```
md1img
spmfw
scp
sspm
dtbo
tee
vendor
vbmeta
boot
system
lk
preloader
```

## 验证方案

### Android_Scatter 文件查看
在编译生成的文件中 , 查看相应的平台的Android_scatter.txt 文件 , 比如在这里的 MT6761_Android_scatter.txt.
```
- partition_index: SYS14
  partition_name: secos_a
  file_name: secos.img
  is_download: true
  type: EXT4_IMG
  linear_start_addr: 0x14400000
  physical_start_addr: 0x14400000
  partition_size: 0x3400000
  region: EMMC_USER
  storage: HW_STORAGE_EMMC
  boundary_check: true
  is_reserved: false
  operation_type: UPDATE
  is_upgradable: false
  empty_boot_needed: false
  reserve: 0x00

```

### 生成文件查看
在编译的生成文件路径下查看 obj/PACKAGING/secos_intermediates 可以看到.
同时查看一下obj/PACKAGING/target_files_intermediates/(full_preoject_target_files)/IMAGES 这个路径是否存在相应的 secos.img 文件 
full_preoject_target_files 是相应的平台的 target_file生成目录



### 系统中查看
系统中验证是否已经添加好一个分区 , 这个分区是否挂载上 ,挂载参数是否正确的的验证方法. 

可以通过查看以下路径, 知道是否建立分区.
```
/dev/block/platform/bootdevice/by-name/

lrwxrwxrwx 1 root root   21 2019-09-06 10:52 boot_a -> /dev/block/mmcblk0p23
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 boot_b -> /dev/block/mmcblk0p36
lrwxrwxrwx 1 root root   20 2019-09-06 10:52 boot_para -> /dev/block/mmcblk0p1
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 dtbo_a -> /dev/block/mmcblk0p24
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 dtbo_b -> /dev/block/mmcblk0p37
lrwxrwxrwx 1 root root   20 2019-09-06 10:52 expdb -> /dev/block/mmcblk0p3
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 flashinfo -> /dev/block/mmcblk0p44
lrwxrwxrwx 1 root root   20 2019-09-06 10:52 frp -> /dev/block/mmcblk0p4
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 gz_a -> /dev/block/mmcblk0p21
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 gz_b -> /dev/block/mmcblk0p34
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 lk_a -> /dev/block/mmcblk0p22
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 lk_b -> /dev/block/mmcblk0p35
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 logo -> /dev/block/mmcblk0p16
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 md1img_a -> /dev/block/mmcblk0p17
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 md1img_b -> /dev/block/mmcblk0p29
lrwxrwxrwx 1 root root   20 2019-09-06 10:52 metadata -> /dev/block/mmcblk0p7
lrwxrwxrwx 1 root root   20 2019-09-06 10:52 nvcfg -> /dev/block/mmcblk0p5
lrwxrwxrwx 1 root root   20 2019-09-06 10:52 nvdata -> /dev/block/mmcblk0p6
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 nvram -> /dev/block/mmcblk0p15
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 otp -> /dev/block/mmcblk0p43
lrwxrwxrwx 1 root root   20 2019-09-06 10:52 para -> /dev/block/mmcblk0p2
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 persist -> /dev/block/mmcblk0p11
lrwxrwxrwx 1 root root   23 2019-09-06 10:52 preloader_a -> /dev/block/mmcblk0boot0
lrwxrwxrwx 1 root root   23 2019-09-06 10:52 preloader_b -> /dev/block/mmcblk0boot1
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 proinfo -> /dev/block/mmcblk0p14
lrwxrwxrwx 1 root root   20 2019-09-06 10:52 protect1 -> /dev/block/mmcblk0p8
lrwxrwxrwx 1 root root   20 2019-09-06 10:52 protect2 -> /dev/block/mmcblk0p9
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 scp_a -> /dev/block/mmcblk0p19
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 scp_b -> /dev/block/mmcblk0p31
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 sec1 -> /dev/block/mmcblk0p13
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 seccfg -> /dev/block/mmcblk0p10
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 secos_a -> /dev/block/mmcblk0p12 // 实际上的块设备, 这个块设备需要当做关键词在log中搜索
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 secos_b -> /dev/block/mmcblk0p33 // 实际上的块设备
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 spmfw_a -> /dev/block/mmcblk0p18
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 spmfw_b -> /dev/block/mmcblk0p30
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 sspm_a -> /dev/block/mmcblk0p20
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 sspm_b -> /dev/block/mmcblk0p32
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 system_a -> /dev/block/mmcblk0p27
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 system_b -> /dev/block/mmcblk0p40
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 tee_a -> /dev/block/mmcblk0p25
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 tee_b -> /dev/block/mmcblk0p38
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 userdata -> /dev/block/mmcblk0p42
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 vbmeta_a -> /dev/block/mmcblk0p28
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 vbmeta_b -> /dev/block/mmcblk0p41
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 vendor_a -> /dev/block/mmcblk0p26
lrwxrwxrwx 1 root root   21 2019-09-06 10:52 vendor_b -> /dev/block/mmcblk0p39


vendor/secos -> /mnt/vendor/secos 
```
### mount 无法查看
我新添加的分区, 在执行 mount 命令的时候, 并没有发现新的分区, 但是到相应的挂在路径下面, 是存在的.
mount 查看并没看到分区, 这个可能和挂载参数有关系．
这个是和挂载参数的最后一列 wait,check,formattable 有关系,添加上这个就可以 mount 直接看到了．
wait,check 也可以查看到
wait,formattable 也可以查看到.
### 检查挂载参数
在有时候 ,我们需要挂载的文件的参数是否正确 , 由以下两个文件可以知道
/vendor/etc/fstab.mt6761  // 挂载的fstab文件
/vendor/etc/fstab.mt8766  // 挂载的fstab文件

### 分区放再userdata 分区的前面
这里是把一个分区放在另一个分区的前面


## 参考文献
### 关键参数说明
[fstab](https://wiki.archlinux.org/index.php/Fstab_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
[fstab](https://wiki.archlinux.org/index.php/Fstab)
[Linux 性能优化之 IO 子系统](http://liaoph.com/linux-system-io/)
[Ext4 Filesystem](https://www.kernel.org/doc/Documentation/filesystems/ext4.txt)


### 添加分区

[MTK平台新增一个分区](https://www.xiezeyang.com/2018/12/08/Filesystem/MTK%E5%B9%B3%E5%8F%B0%E6%96%B0%E5%A2%9E%E4%B8%80%E4%B8%AA%E5%88%86%E5%8C%BA/)
MTK的FAQ19053
