# SPDX-License-Identifier: Apache-2.0
#
# Copyright (C) 2021, ARM Limited and contributors.
#
# Licensed under the Apache License, Version 2.0 (the "License"); you may
# not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
# WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

ifneq ($(KERNELRELEASE),)
	MODULE_SRC ?= $(src)
	MODULE_OBJ := $(obj)
	KERNEL_SRC ?= $(srctree)
else
	MODULE_SRC ?= $(PWD)
	MODULE_OBJ := $(PWD)
	KERNEL_SRC ?= /lib/modules/`uname -r`/build
endif

VMLINUX_H = $(MODULE_OBJ)/vmlinux.h
SYMBOL_NAMESPACES_H = $(MODULE_OBJ)/symbol_namespaces.h

# kbuild part of makefile. Only Kbuild-related targets should be used here to
# avoid any sort of clash.
ifneq ($(KERNELRELEASE),)

LISA_KMOD_NAME ?= sched_tp
obj-m := $(LISA_KMOD_NAME).o
$(LISA_KMOD_NAME)-y := main.o tp.o wq.o features.o pixel6.o
ldflags-y += -T $(M)/features.lds

clean-files := vmlinux.h

# -fno-stack-protector is needed to possibly undefined __stack_chk_guard symbol
ccflags-y = -I$(MODULE_SRC) -std=gnu11 -fno-stack-protector -Wno-declaration-after-statement

VMLINUX_TXT = $(MODULE_SRC)/private_types.txt

# Can be either a kernel image built with DWARF debug info, or the BTF blob
# found at /sys/kernel/btf/vmlinux
_BTF_VMLINUX = $(MODULE_SRC)/vmlinux
_DWARF_VMLINUX = $(KERNEL_SRC)/vmlinux
ifneq ("$(wildcard $(_BTF_VMLINUX))","")
    VMLINUX := $(_BTF_VMLINUX)
else
    VMLINUX := $(_DWARF_VMLINUX)
endif

$(VMLINUX_H): $(VMLINUX_TXT) $(VMLINUX)
# Some options are not upstream (yet) but they have all be published on the
# dwarves mailing list
#
# Options:
# -F btf,dwarf: Use BTF first
# -E: Expand nested type definitions
# --suppress_force_paddings: Remove the "artificial" padding members pahole adds
#   to make padding more visible. They are not always valid C syntax and can
#   break build
# --skip_missing: Keep going if one of the types is not found
# --forward_decl: (non upstream) Include forward declaration of union and
#   struct. Note that it is not currently used as it dumps the declaration of
#   all types. Unfortunately, some android kernels have multiple definitions of
#   the same type (struct map_info/union map_info) which renders the header
#   invalid. Since it's not a type we are interested in, we don't ask pahole to
#   generate forward_decl and instead do it manually with sed.
# --expand_types_once: Only expand a given type once, to avoid type redefinition
#   (C does not care about nesting types, there is a single namespace).
# --expanded_prefix: Add a prefix to all the types that are expanded by -E. This
#   avoids conflicting with type definitions that would come from public kernel
#   headers, while still allowing easy attribute access, anonymous types etc.
	pahole -F btf,dwarf -E --suppress_force_paddings --show_only_data_members --skip_missing --expand_types_once --expanded_prefix "KERNEL_PRIVATE_" -C file://$(VMLINUX_TXT) $(VMLINUX) > _header
# Strip comments to avoid matching them with the sed regex.
	$(CC) -P -E - < _header > _header_no_comment
# Create forward declaration of every type
	sed -r -n 's/.*(struct|union|enum) ([0-9a-zA-Z_]*) .*/\1 \2;/p' _header_no_comment | sort -u > _fwd_decl
# Create TYPED_DEFINED_struct_foo macros for every type, so the client code can
# check whether a given type exists before making use of it
	sed -r -n 's/.*(struct|union|enum) ([0-9a-zA-Z_]*) .*/#define TYPE_DEFINED_\1_\2/p' _header_no_comment | sort -u >> _fwd_decl
	cat _fwd_decl _header > $@
# cat $@

# Some kernels require the use of MODULE_IMPORT_NS() before using symbols that are part of the given namespace:
# https://docs.kernel.org/core-api/symbol-namespaces.html
# Unfortunately, Android kernels seem to define their own namespaces for GKI, so
# in order to avoid issues and work on any kernel, we simply attempt to list all
# of the namespaces this kernel seems to rely on by looking at the sources.
# We could use Module.symvers file, but it can only be generated when building a
# kernel so it would be way to slow.
# There does not seem to be any other source for the info in e.g. sysfs or
# procfs, so we rely on that hack for now.
$(SYMBOL_NAMESPACES_H):
	find $(KERNEL_SRC) '(' -name '*.c' -o -name '*.h' ')' -print0 | xargs -0 sed -n 's/MODULE_IMPORT_NS([^;]*;/\0/p' | sort -u > $@

# Make all object files depend on the generated sources
$(addprefix $(MODULE_OBJ)/,$($(LISA_KMOD_NAME)-y)): $(VMLINUX_H) $(SYMBOL_NAMESPACES_H)

# Non-Kbuild part
else

.PHONY: all build install clean

all: install

build:
	$(MAKE) -C $(KERNEL_SRC) M=$(MODULE_SRC) modules

install: build
	$(MAKE) -C $(KERNEL_SRC) M=$(MODULE_SRC) modules_install

clean:
	rm -f $(VMLINUX_H) $(SYMBOL_NAMESPACES_H)

endif
