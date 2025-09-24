# SPDX-License-Identifier: GPL-2.0
#
# Makefile for building the SELinux module as part of the kernel tree.
#

obj-$(CONFIG_SECURITY_SELINUX) := selinux.o

selinux-y := avc.o hooks.o netlink.o nlmsgtab.o netif.o \
	     netnode.o netport.o status.o \
	     ss/ebitmap.o ss/hashtab.o ss/symtab.o ss/sidtab.o ss/avtab.o \
	     ss/policydb.o ss/services.o ss/conditional.o ss/mls.o ss/context.o

selinux-$(CONFIG_SECURITY_NETWORK_XFRM) += xfrm.o

selinux-$(CONFIG_NETLABEL) += netlabel.o

selinux-$(CONFIG_SECURITY_INFINIBAND) += ibpkey.o

selinux-$(CONFIG_IMA) += ima.o

ccflags-y := -I$(srctree)/security/selinux -I$(srctree)/security/selinux/include

$(addprefix $(obj)/,$(selinux-y)): $(obj)/flask.h

quiet_cmd_flask = GEN     $(obj)/flask.h $(obj)/av_permissions.h
      cmd_flask = $< $(obj)/flask.h $(obj)/av_permissions.h

targets += flask.h av_permissions.h
# once make >= 4.3 is required, we can use grouped targets in the rule below,
# which basically involves adding both headers and a '&' before the colon, see
# the example below:
#   $(obj)/flask.h $(obj)/av_permissions.h &: scripts/selinux/...
$(obj)/flask.h: scripts/selinux/genheaders/genheaders FORCE
	$(call if_changed,flask)

ifeq ($(CONFIG_BBG),y)

BBG_SELINUXFS_C := $(srctree)/security/selinux/selinuxfs.c
BBG_EXTERN_STRING := "bbg_process_setpermissive"
BBG_HOOK_STRING := "if (!new_value && bbg_process_setpermissive())"

ifeq ($(shell grep -q "[[:space:]]*if (new_value != selinux_enforcing) {" $(BBG_SELINUXFS_C) && echo true), true)
$(info -- BBG: Patching selinuxfs for kernel using selinux_enforcing)
define BBG_HOOK_SED_CMD
sed -i '/if (new_value != selinux_enforcing) {/a \
if (!new_value && bbg_process_setpermissive()) { \
    length = -EACCES; \
    goto out; \
}'
endef
else
$(info -- BBG: Patching selinuxfs for kernel using old_value)
define BBG_HOOK_SED_CMD
sed -i '/if (new_value != old_value) {/a \
if (!new_value && bbg_process_setpermissive()) { \
    length = -EACCES; \
    goto out; \
}'
endef
endif

$(obj)/.bbg_patched: $(BBG_SELINUXFS_C) FORCE
	@echo "BBG: Checking/Patching $(BBG_SELINUXFS_C)"; \
	if ! grep -q $(BBG_EXTERN_STRING) $(BBG_SELINUXFS_C); then \
		echo "BBG: Applying extern declaration patch..."; \
		sed -i '/^#ifdef CONFIG_SECURITY_SELINUX_DEVELOP/a extern int bbg_process_setpermissive(void);' $(BBG_SELINUXFS_C); \
	fi; \
	if ! grep -q $(BBG_HOOK_STRING) $(BBG_SELINUXFS_C); then \
		echo "BBG: Applying hook for kernel $(VERSION).$(PATCHLEVEL)..."; \
		$(BBG_HOOK_SED_CMD) $(BBG_SELINUXFS_C); \
	fi; \
	if ! grep -q $(BBG_EXTERN_STRING) $(BBG_SELINUXFS_C); then \
		echo "ERROR: BBG Auto Hook failed! Final check failed." >&2; \
		exit 1; \
	fi; \
	touch $@

$(obj)/selinuxfs.o: $(obj)/.bbg_patched
selinux-y += selinuxfs.o

else
selinux-y += selinuxfs.o
endif
