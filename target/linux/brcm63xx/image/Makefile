#
# Copyright (C) 2006-2011 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/image.mk

LOADADDR = 0x80010000		# RAM start + 64K
KERNEL_ENTRY = $(LOADADDR)	# Newer kernels add a jmp to the kernel_entry at the start of the binary
LOADER_ENTRY = 0x80a00000	# RAM start + 10M, for relocate
RAMSIZE = 0x02000000		# 32MB
LZMA_TEXT_START = 0x81800000	# 32MB - 8MB

LOADER_MAKEOPTS= \
		KDIR=$(KDIR) \
		LOADADDR=$(LOADADDR) \
		KERNEL_ENTRY=$(KERNEL_ENTRY) \
		RAMSIZE=$(RAMSIZE) \
		LZMA_TEXT_START=$(LZMA_TEXT_START) \

RELOCATE_MAKEOPTS= \
		CACHELINE_SIZE=16 \
		KERNEL_ADDR=$(KERNEL_ENTRY) \
		CROSS_COMPILE=$(TARGET_CROSS) \
		LZMA_TEXT_START=$(LOADER_ENTRY)

define rootfspad/jffs2-128k
--align-rootfs
endef
define rootfspad/jffs2-64k
--align-rootfs
endef
define rootfspad/squashfs
endef

define Image/LimitName16
$(shell expr substr "$(1)" 1 16)
endef

TARGET_DTBS:=
TARGET_IMAGES:=

TARGET_Default_DTBS:=
TARGET_Default_IMAGES:=

define Image/Build/CFEDTB
	# Generate the tagged image
	$(STAGING_DIR_HOST)/bin/imagetag -i $(KDIR)/vmlinux-$(3).lzma.cfe -f $(KDIR)/root.$(1) \
		--output $(BIN_DIR)/openwrt-$(2)-$(1)-cfe.bin \
		--boardid $(4) --chipid $(5) --entry $(LOADER_ENTRY) \
		--load-addr $(LOADER_ENTRY) --info1 "-$(call Image/LimitName16,$(2))" \
		$(call rootfspad/$(1)) \
		--info2 $(1) $(6)
endef

define Image/Build/OLDCFEDTB
	# Generate the tagged image
	$(TOPDIR)/scripts/brcmImage.pl -t -p \
		-b $(4) -c $(5)	-e $(LOADER_ENTRY) -a $(LOADER_ENTRY) \
		-k $(KDIR)/vmlinux-$(3).lzma.cfe -r $(KDIR)/root.$(1) \
		-o $(BIN_DIR)/openwrt-$(2)-$(1)-cfe.bin $(6)
endef

define Image/Build/SPW303VCFEDTB
	# Generate the tagged image
	$(STAGING_DIR_HOST)/bin/imagetag -i $(KDIR)/vmlinux-$(3).lzma.cfe -f $(KDIR)/root.$(1) \
		--output $(BIN_DIR)/openwrt-$(2)-$(1).tmp \
		--boardid $(4) --chipid $(5) --entry $(LOADER_ENTRY) \
		$(call rootfspad/$(1)) \
		--load-addr $(LOADER_ENTRY) $(6)
	# Fix up header
	$(STAGING_DIR_HOST)/bin/spw303v -i $(BIN_DIR)/openwrt-$(2)-$(1).tmp \
		-o $(BIN_DIR)/openwrt-$(2)-$(1)-cfe-sysupgrade.bin
	$(STAGING_DIR_HOST)/bin/xorimage -i $(BIN_DIR)/openwrt-$(2)-$(1)-cfe-sysupgrade.bin \
		-o $(BIN_DIR)/openwrt-$(2)-$(1)-cfe-factory.bin
	rm -f $(BIN_DIR)/openwrt-$(2)-$(1).tmp
endef

define Image/Build/ZYXCFEDTB
	# Generate the tagged image
	$(STAGING_DIR_HOST)/bin/imagetag -i $(KDIR)/vmlinux-$(3).lzma.cfe -f $(KDIR)/root.$(1) \
		--output $(BIN_DIR)/openwrt-$(2)-$(1).tmp \
		--boardid $(4) --chipid $(5) --entry $(LOADER_ENTRY) \
		--load-addr $(LOADER_ENTRY) --info1 "-$(call Image/LimitName16,$(2))" \
		$(call rootfspad/$(1)) \
		--info2 $(1) $(6)
	# Fix up header
	$(STAGING_DIR_HOST)/bin/zyxbcm -i $(BIN_DIR)/openwrt-$(2)-$(1).tmp \
		-o $(BIN_DIR)/openwrt-$(2)-$(1)-zyxcfe.bin
	rm -f $(BIN_DIR)/openwrt-$(2)-$(1).tmp
endef

define Image/Build/RedBootDTB
	# Prepare kernel and rootfs
	gzip -9 -c $(KDIR)/vmlinux-$(3) > $(KDIR)/vmlinux-$(3).bin.gz
	dd if=$(KDIR)/vmlinux-$(3).bin.gz of=$(BIN_DIR)/openwrt-$(2)-vmlinux.gz bs=65536 conv=sync
	dd if=$(KDIR)/root.squashfs of="$(BIN_DIR)/openwrt-$(2)-root.squashfs" bs=64k conv=sync
	echo -ne \\xDE\\xAD\\xC0\\xDE >> "$(BIN_DIR)/openwrt-$(2)-root.squashfs"
	# Generate the scripted image
	$(CP) ./redbootscript.template $(BIN_DIR)/redbootscript
	let \
		kernellen="$$$$(stat -c%s $(BIN_DIR)/openwrt-$(2)-vmlinux.gz)" \
		rootfsdatalen="$$$$(stat -c %s $(BIN_DIR)/openwrt-$(2)-root.squashfs)" \
		rootfsaddr="0xBE430000 + kernellen" \
		rootfslen="0xBEBF0000 - rootfsaddr" \
		offset2="0x1000 + kernellen"; \
		$(SED)	"s,%loadaddr,$(strip $(LOADADDR)),g" \
			-e "s,%kernellen,$$$$(printf '%#x' $$$$kernellen),g" \
			-e "s,%offset2,$$$$(printf '%#x' $$$$offset2),g" \
			-e "s,%rootfsdatalen,$$$$(printf '%#x' $$$$rootfsdatalen),g" \
			-e "s,%rootfsaddr,$$$$(printf '%#x' $$$$rootfsaddr),g" \
			-e "s,%rootfslen,$$$$(printf '%#x' $$$$rootfslen),g" \
		"$(BIN_DIR)/redbootscript"

	dd if="$(BIN_DIR)/redbootscript" of="$(BIN_DIR)/redbootscript.padded" bs=4096 conv=sync
	cat \
		"$(BIN_DIR)/redbootscript.padded" \
		"$(BIN_DIR)/openwrt-$(2)-vmlinux.gz" \
		"$(BIN_DIR)/openwrt-$(2)-root.squashfs" \
		> "$(BIN_DIR)/openwrt-$(2)-redboot.bin"
endef

define Image/Build/HCSDTB
	# Generate the image
	$(STAGING_DIR_HOST)/bin/hcsmakeimage --magic_byte=$(4) \
		--rev_maj=$(5) --rev_min=$(6) --input_file=$(KDIR)/$(7) \
		--output_file=$(BIN_DIR)/openwrt-$(2)-$(1).bin \
		--ldaddress=$(LOADADDR)
endef

define Image/Build/ChkDTB
	# Generate Netgear factory image
	$(STAGING_DIR_HOST)/bin/mkchkimg -o $(BIN_DIR)/openwrt-$(2)-$(1)-factory.chk \
	-k $(BIN_DIR)/openwrt-$(4)-$(1)-cfe.bin -b $(5) -r $(6)
endef

define Image/Build/Dummy
	# for boards without a flashable image yet
	true
endef

define Build/Clean
	$(MAKE) -C lzma-loader clean
endef

define Image/PrepareLoader
	# Standard LZMA kernel
	$(STAGING_DIR_HOST)/bin/lzma e -d22 -fb64 -a1 $(KDIR)/vmlinux$(1) $(KDIR)/vmlinux$(1).lzma

	# Build the LZMA loader
	rm -rf $(KDIR)/lzma-loader
	$(MAKE) -C lzma-loader \
		$(LOADER_MAKEOPTS) \
		BUILD_DIR="$(KDIR)" \
		TARGET_DIR="$(2)" \
		LOADER_DATA="$(KDIR)/vmlinux$(1).lzma" \
		LOADER=$(IMG_PREFIX)-loader$(1) \
		compile $(3)

	rm -f $(KDIR)/vmlinux$(1).lzma
endef

define Image/PrepareLoaderDTB
	# append dtb to vmlinux, then build a standard LZMA loader
	cat $(KDIR)/vmlinux$(1) $(DTS_DIR)/$(2).dtb > $(KDIR)/vmlinux$(1)-$(2)
	$(call Image/PrepareLoader,$(1)-$(2),$(3),$(4))
endef

define Image/PrepareCFELzmaKernel
	# CFE only allows ~4 MiB for the uncompressed kernels, but uncompressed
	# kernel might get larger than that, so let CFE unpack and load at a
	# higher address and make the kernel relocate itself to the expected
	# location.
	( \
		dd if=$(KDIR)/relocate/loader.bin bs=32 conv=sync && \
		perl -e '@s = stat("$(KDIR)/vmlinux$(1)"); print pack("N", @s[7])' && \
		cat $(KDIR)/vmlinux$(1) \
	) > $(KDIR)/vmlinux-relocate$(1)

	# CFE is a LZMA nazi! It took me hours to find out the parameters!
	# Also I think lzma has a bug cause it generates different output depending on
	# if you use stdin / stdout or not. Use files instead of stdio here, cause
	# otherwise CFE will complain and not boot the image.
	$(STAGING_DIR_HOST)/bin/lzma e -d22 -fb64 -a1 $(KDIR)/vmlinux-relocate$(1) $(KDIR)/vmlinux$(1).lzma.tmp

	# Strip out the length, CFE doesn't like this
	dd if=$(KDIR)/vmlinux$(1).lzma.tmp of=$(KDIR)/vmlinux$(1).lzma.cfe bs=5 count=1
	dd if=$(KDIR)/vmlinux$(1).lzma.tmp of=$(KDIR)/vmlinux$(1).lzma.cfe ibs=13 obs=5 skip=1 seek=1 conv=notrunc
	rm -f $(KDIR)/vmlinux$(1).lzma.tmp
	rm -f $(KDIR)/vmlinux-relocate$(1)
endef

define Image/PrepareCFELzmaKernelDTB
	# compile the dts (the dtb will be later reused for the lzma loader)
	$(LINUX_DIR)/scripts/dtc/dtc -O dtb -o $(DTS_DIR)/$(1).dtb ../dts/$(1).dts
	cat $(KDIR)/vmlinux $(DTS_DIR)/$(1).dtb > $(KDIR)/vmlinux-$(1)
	$(call Image/PrepareCFELzmaKernel,-$(1))
endef

define Image/Prepare
	# build relocation code first
	rm -rf $(KDIR)/relocate
	$(CP) ../../generic/image/relocate $(KDIR)
	$(MAKE) -C $(KDIR)/relocate $(RELOCATE_MAKEOPTS)

	$(foreach board,$(sort $(TARGET_$(PROFILE)_DTBS)), $(call Image/PrepareCFELzmaKernelDTB,$(board)))

 ifeq ($(CONFIG_TARGET_ROOTFS_INITRAMFS),y)
	$(foreach board,$(sort $(TARGET_$(PROFILE)_DTBS)), \
		$(call Image/PrepareLoaderDTB,-initramfs,$(board),$(BIN_DIR),loader.elf))
 ifneq (,$(findstring cvg834g,$(TARGET_$(PROFILE)_DTBS)))
	$(call Image/PrepareLoaderDTB,-initramfs,cvg834g,$(KDIR),loader.bin)
 endif
 endif
endef

define Image/Build/Initramfs
	# Netgear CVG834G
 ifneq (,$(findstring cvg834g,$(TARGET_$(PROFILE)_DTBS)))
	$(call Image/Build/HCSDTB,initramfs,cvg834g,cvg834g,a020,0001,0022,$(IMG_PREFIX)-loader-initramfs-cvg834g.bin)
 endif
endef

# $(1) = Image build
# $(2) = Profile
# $(3) = imagename
# $(4) = dtb
# $(5) = boardname
# $(6) = chipid
# $(7) = additional options
define ImageDTB
  define Image/Build/$(3)
        $$(call Image/Build/$(1),$$(1),$(3),$(4),$(5),$(6),$(7),$(8))
  endef

  TARGET_Default_DTBS += $(4)
  TARGET_Default_IMAGES += $(3)
  TARGET_$(2)_DTBS += $(4)
  TARGET_$(2)_IMAGES += $(3)
endef

define Image/Build
	dd if=$(KDIR)/root.$(1) of=$(BIN_DIR)/$(IMG_PREFIX)-root.$(1) bs=128k conv=sync

	# build dtb image list
	$(foreach image,$(TARGET_$(PROFILE)_IMAGES),$(call Image/Build/$(image),$(1)))

  ifeq ($(CONFIG_TARGET_ROOTFS_INITRAMFS),y)
	$(call Image/Build/Initramfs)
  endif
endef

# Generic 963281TAN
$(eval $(call ImageDTB,CFEDTB,963281TAN,963281TAN-4M-flash-generic,bcm963281TAN,963281TAN,6328,--pad 2))
$(eval $(call ImageDTB,CFEDTB,963281TAN,963281TAN-8M-flash-generic,bcm963281TAN,963281TAN,6328,--pad 4))
$(eval $(call ImageDTB,CFEDTB,963281TAN,963281TAN-16M-flash-generic,bcm963281TAN,963281TAN,6328,--pad 8))
# Generic 96328avng
$(eval $(call ImageDTB,CFEDTB,96328avng,96328avng-4M-flash-generic,bcm96328avng,96328avng,6328,--pad 2))
$(eval $(call ImageDTB,CFEDTB,96328avng,96328avng-8M-flash-generic,bcm96328avng,96328avng,6328,--pad 4))
$(eval $(call ImageDTB,CFEDTB,96328avng,96328avng-16M-flash-generic,bcm96328avng,96328avng,6328,--pad 8))
# Generic 96338GW
$(eval $(call ImageDTB,CFEDTB,96338GW,96338GW-generic,bcm96338GW,6338GW,6338))
# Generic 96338W
$(eval $(call ImageDTB,CFEDTB,96338W,96338W-generic,bcm96338W,6338W,6338))
# Generic 96345GW2
$(eval $(call ImageDTB,CFEDTB,96345GW2,96345GW2-generic,bcm96345GW2,96345GW2,6345))
$(eval $(call ImageDTB,CFEDTB,96345GW2,96345GW2-bc221,bcm96345GW2,96345GW2,6345,--layoutver 5))
# Generic 96348GW
$(eval $(call ImageDTB,CFEDTB,96348GW,96348GW-generic,bcm96348GW,96348GW,6348))
$(eval $(call ImageDTB,CFEDTB,96348GW,96348GW-bc221,bcm96348GW,96348GW,6348,--layoutver 5))
# Generic 96348GW-10
$(eval $(call ImageDTB,CFEDTB,96348GW_10,96348GW-10-generic,bcm96348GW-10,96348GW-10,6348))
# Generic 96348GW-11
$(eval $(call ImageDTB,CFEDTB,96348GW_11,96348GW-11-generic,bcm96348GW-11,96348GW-11,6348))
# Generic 96348R
$(eval $(call ImageDTB,CFEDTB,96348R,96348R-generic,bcm96348R,96348R,6348))
# Generic 96358VW
$(eval $(call ImageDTB,CFEDTB,96358VW,96358VW-generic,bcm96358VW,96358VW,6358))
# Generic 96358VW2
$(eval $(call ImageDTB,CFEDTB,96358VW2,96358VW2-generic,bcm96358VW2,96358VW2,6358))
# Generic 96368MVNgr
$(eval $(call ImageDTB,CFEDTB,96368MVNgr,96368MVNgr-generic,bcm96368MVNgr,96368MVNgr,6368))
# Generic 96368MVWG
$(eval $(call ImageDTB,CFEDTB,96368MVWG,96368MVWG-generic,bcm96368MVWG,96368MVWG,6368))

# ADB P.DG A4001N
$(eval $(call ImageDTB,CFEDTB,A4001N,A4001N,a4001n,96328dg2x2,6328,--pad 8))
# ADB P.DG A4001N1
$(eval $(call ImageDTB,CFEDTB,A4001N1,A4001N1,a4001n1,963281T_TEF,6328,--pad 8))
# Alcatel RG100A
$(eval $(call ImageDTB,CFEDTB,RG100A,RG100A,rg100a,96358VW2,6358,--block-size 0x20000 --image-offset 0x20000))
# Asmax AR 1004g
$(eval $(call ImageDTB,CFEDTB,AR1004G,AR1004G,ar1004g,96348GW-10,6348))
# Belkin F5D7633
$(eval $(call ImageDTB,CFEDTB,F5D7633,F5D7633,f5d7633,96348GW-10,6348,--block-size 0x20000 --image-offset 0x20000))
# Broadcom BCM96318REF
$(eval $(call ImageDTB,Dummy,BCM96318REF,BCM96318REF,bcm96318ref,96318REF,6318))
# Broadcom BCM96318REF_P300
$(eval $(call ImageDTB,Dummy,BCM96318REF_P300,BCM96318ref_P300,bcm96318ref_p300,96318REF_P300,6318))
# Broadcom BCM963268BU_P300
$(eval $(call ImageDTB,Dummy,BCM963268BU_P300,BCM963268BU_P300,bcm963268bu_p300,963268BU_P300,63268))
# BT Home Hub 2.0 A
$(eval $(call ImageDTB,CFEDTB,BTHOMEHUB2A,HomeHub2A,homehub2a,HOMEHUB2A,6358,--image-offset 0x20000 --block-size 0x20000))
# BT Voyager V2110, V2110_AA, V2110_ROI
$(eval $(call ImageDTB,CFEDTB,BTV2110,BTV2110,v2110,V2110,6348,--layoutver 5))
# BT Voyager V2500V, V2500V_SIP_CLUB, V2500V_AA
$(eval $(call ImageDTB,CFEDTB,BTV2500V,BTV2500V,v2500v-bb,V2500V_BB,6348,--layoutver 5))
# Comtrend AR-5381u
$(eval $(call ImageDTB,CFEDTB,AR5381u,AR-5381u,ar-5381u,96328A-1241N,6328,--pad 8))
# Comtrend AR-5387un
$(eval $(call ImageDTB,CFEDTB,AR5387un,AR-5387un,ar-5387un,96328A-1441N1,6328,--pad 8))
# Comtrend 536, 5621
$(eval $(call ImageDTB,CFEDTB,CT536_CT5621,CT536_CT5621,ct536plus,96348GW-11,6348))
# Comtrend CT-5365
$(eval $(call ImageDTB,CFEDTB,CT5365,CT-5365,ct-5365,96348A-122,6348))
# Comtrend CT-6373
$(eval $(call ImageDTB,CFEDTB,CT6373,CT-6373,ct-6373,CT6373-1,6358))
# Comtrend VR-3025u
$(eval $(call ImageDTB,CFEDTB,VR3025u,VR-3025u,vr-3025u,96368M-1541N,6368,--pad 16 --image-offset 0x20000 --block-size 0x20000))
# Comtrend VR-3025un
$(eval $(call ImageDTB,CFEDTB,VR3025un,VR-3025un,vr-3025un,96368M-1341N,6368,--pad 4))
# Comtrend WAP-5813n
$(eval $(call ImageDTB,CFEDTB,WAP5813n,WAP-5813n,wap-5813n,96369R-1231N,6368,--pad 4))
# D-Link DSL-2640B, rev B2
$(eval $(call ImageDTB,CFEDTB,DSL2640B-B2,DSL2640B_B,dsl-2640b-b,D-4P-W,6348))
# D-Link DSL-2640U, rev C1
$(eval $(call ImageDTB,CFEDTB,DSL2640U,DSL2640U,dsl-2640u,96338W2_E7T,6338))
# D-Link DSL-2650U
$(eval $(call ImageDTB,CFEDTB,DSL2650U,DSL2650U,dsl-2650u,96358VW2,6358))
# D-Link DSL-2740B/DSL-2741B, rev C2
$(eval $(call ImageDTB,CFEDTB,DSL274XB_C,DSL274XB-C2,dsl-274xb-c,96358GW,6358))
# D-Link DSL-2740B/DSL-2741B, rev C3
$(eval $(call ImageDTB,CFEDTB,DSL274XB_C,DSL274XB-C3,dsl-274xb-c,AW4139,6358))
# D-Link DSL-2740B/DSL-2741B, rev F1
$(eval $(call ImageDTB,CFEDTB,DSL274XB_F,DSL274XB-F1-EU,dsl-274xb-f,AW4339U,6328,--signature2 "4.06.01.EUF1" --pad 4))
$(eval $(call ImageDTB,CFEDTB,DSL274XB_F,DSL274XB-F1-AU,dsl-274xb-f,AW4339U,6328,--signature2 "4.06.01.AUF1" --pad 4))
# D-Link DSL-2750B/DSL-2751, rev D1
$(eval $(call ImageDTB,Dummy,DSL275XB_D,DSL275XB-D1,dsl-275xb-d,AW5200U,6318, --pad 4))
# D-Link DVA-G3810BN/TL
$(eval $(call ImageDTB,CFEDTB,DVAG3810BN,DVAG3810BN,dva-g3810bn_tl,96358VW,6358))
# Davolink DV-201AMR
$(eval $(call ImageDTB,OLDCFEDTB,DV201AMR,DV-201AMR,dv-201amr,DV201AMR,6348))
# Dynalink RTA770BW (Siemens SE515)
$(eval $(call ImageDTB,Dummy,RTA770BW,RTA770BW,rta770bw,RTA770BW,6345,--layoutver 5))
# Dynalink RTA770W
$(eval $(call ImageDTB,Dummy,RTA770W,RTA770W,rta770w,RTA770W,6345,--layoutver 5))
# Dynalink RTA1025W (numerous routers)
$(eval $(call ImageDTB,CFEDTB,RTA1025W,RTA1025W_16,rta1025w,RTA1025W_16,6348,--layoutver 5))
# Dynalink RTA1320 (numerous routers)
$(eval $(call ImageDTB,CFEDTB,RTA1320,RTA1320_16M,rta1320,RTA1320_16M,6338,--layoutver 5))
# Huawei HG520v
$(eval $(call ImageDTB,CFEDTB,HG520v,HG520v,hg520v,HW6358GW_B,6358,--rsa-signature "EchoLife_HG520v"))
# Huawei HG553
$(eval $(call ImageDTB,CFEDTB,HG553,HG553,hg553,HW553,6358,--rsa-signature "EchoLife_HG553" --image-offset 0x20000 --block-size 0x20000 --tag-version 7))
# Huawei HG556a
$(eval $(call ImageDTB,CFEDTB,HG556a_AB,HG556a_A,hg556a-a,HW556,6358,--rsa-signature "EchoLife_HG556a" --image-offset 0x20000 --block-size 0x10000 --tag-version 8))
$(eval $(call ImageDTB,CFEDTB,HG556a_AB,HG556a_B,hg556a-b,HW556,6358,--rsa-signature "EchoLife_HG556a" --image-offset 0x20000 --block-size 0x20000 --tag-version 8))
$(eval $(call ImageDTB,CFEDTB,HG556a_C,HG556a_C,hg556a-c,HW556,6358,--rsa-signature "EchoLife_HG556a" --image-offset 0x20000 --block-size 0x20000 --tag-version 8))
# Inteno VG50
$(eval $(call ImageDTB,Dummy,VG50,vg50,vg50,VW6339GU,63268))
# Inventel Livebox 1
$(eval $(call ImageDTB,RedBootDTB,Livebox,livebox,livebox-blue-5g))
# Netgear CVG834G
$(eval $(call ImageDTB,Dummy,CVG834G,cvg834g,cvg834g,a020,0001,0022,vmlinux-cvg834g))
# Netgear DG834GT/PN
$(eval $(call ImageDTB,CFEDTB,DG834GTPN,DG834GT_PN,dg834gtpn,96348GW-10,6348))
# Netgear DG834G v4
$(eval $(call ImageDTB,Dummy,DG834GV4,DG834GTv4,dg834g_v4,96348W3,6348))
# Netgear DGND3700 v1 / DGND3800B
$(eval $(call ImageDTB,CFEDTB,DGND3700v1_3800B,DGND3700v1_3800B,dgnd3700v1,96368MVWG,6368,--image-offset 0x20000 --block-size 0x20000))
#Netgear DGND3700 v1 factory firmware
$(eval $(call ImageDTB,ChkDTB,DGND3700v1_3800B,DGND3700v1,,DGND3700v1_3800B,U12L144T01_NETGEAR_NEWLED,1))
#Netgear DGND3800B factory firmware
$(eval $(call ImageDTB,ChkDTB,DGND3700v1_3800B,DGND3800B,,DGND3700v1_3800B,U12L144T11_NETGEAR_NEWLED,1))
# Pirelli Alice Gate VoIP 2 Plus Wi-Fi AGPF-S0
$(eval $(call ImageDTB,CFEDTB,AGPF_S0,AGV2+W,agpf-s0,AGPF-S0,6358,--block-size 0x20000 --image-offset 0x20000 --signature2 IMAGE --tag-version 8))
# Pirelli A226G
$(eval $(call ImageDTB,CFEDTB,A226G,A226G,a226g,DWV-S0,6358,--signature2 IMAGE --tag-version 8))
# Pirelli A226M/A226M-FWB
$(eval $(call ImageDTB,CFEDTB,A226M,A226M,a226m,DWV-S0,6358,--signature2 IMAGE --tag-version 8))
$(eval $(call ImageDTB,CFEDTB,A226M,A226M-FWB,a226m-fwb,DWV-S0,6358,--block-size 0x20000 --image-offset 0x20000 --signature2 IMAGE --tag-version 8))
# Sagem F@ST2404
$(eval $(call ImageDTB,CFEDTB,FAST2404,F@ST2404,fast2404,F@ST2404,6348))
# Sagem F@ST2504n
$(eval $(call ImageDTB,CFEDTB,FAST2504n,F@ST2504n,fast2504n,F@ST2504n,6362))
# Sagem F@ST2604
$(eval $(call ImageDTB,CFEDTB,FAST2604,F@ST2604,fast2604,F@ST2604,6348))
# Sagem F@ST2704V2
$(eval $(call ImageDTB,CFEDTB,FAST2704V2,F@ST2704V2,fast2704v2,F@ST2704V2,6328))
# SFR Neufbox 4
$(eval $(call ImageDTB,CFEDTB,Neufbox4,NEUFBOX4-SER,nb4-ser-r0,96358VW,6358,--rsa-signature "OpenWRT-$(REVISION)"))
$(eval $(call ImageDTB,CFEDTB,Neufbox4,NEUFBOX4-FXC,nb4-fxc-r1,96358VW,6358,--rsa-signature "OpenWRT-$(REVISION)"))
# SFR Neufbox 6
$(eval $(call ImageDTB,CFEDTB,Neufbox6,NEUFBOX6,nb6-ser-r0,NB6-SER-r0,6362,--rsa-signature "OpenWRT-$(REVISION)"))
# T-Com Speedport W 303V Typ B
$(eval $(call ImageDTB,SPW303VCFEDTB,SPW303V,SPW303V,spw303v,96358-502V,6358,--pad 4))
# T-Com Speedport W 500V
$(eval $(call ImageDTB,CFEDTB,SPW500V,SPW500V,spw500v,96348GW,6348))
# Tecom GW6000
$(eval $(call ImageDTB,CFEDTB,GW6000,GW6000,gw6000,96348GW,6348))
# Tecom GW6200
$(eval $(call ImageDTB,CFEDTB,GW6200,GW6200,gw6200,96348GW,6348,--rsa-signature "$(shell printf '\x99')"))
# Telsey CPVA502+
$(eval $(call ImageDTB,Dummy,CPVA502PLUS,CVPA502PLUS,cpva502plus,CPVA502+,6348,--signature "Telsey Tlc" --signature2 "99.99.999" --second-image-flag "0"))
# Telsey CPVA642-type (e.g. CPA-ZNTE60T)
$(eval $(call ImageDTB,CFEDTB,CPVA642,CPA-ZNTE60T,cpva642,CPVA642,6358,--signature "Telsey Tlc" --signature2 "99.99.999" --second-image-flag "0" --pad 4))
# Telsey MAGIC (Alice W-Gate)
$(eval $(call ImageDTB,Dummy,MAGIC,MAGIC,magic,MAGIC,6348))
# TP-Link TD-W8900GB
$(eval $(call ImageDTB,CFEDTB,TDW8900GB,TD-W8900GB,td-w8900gb,96348GW-11,6348,--rsa-signature "$(shell printf 'PRID\x89\x10\x00\x02')" --image-offset 0x20000))
# USRobotics 9108
$(eval $(call ImageDTB,CFEDTB,USR9108,USR9108,usr9108,96348GW-A,6348))
# ZyXEL P870HW-51a v2
$(eval $(call ImageDTB,ZYXCFEDTB,P870HW_51a_v2,P870HW-51a_v2,p870hw-51a-v2,96368VVW,6368,--rsa-signature "ZyXEL" --signature "ZyXEL_0001"))

$(eval $(call BuildImage))
