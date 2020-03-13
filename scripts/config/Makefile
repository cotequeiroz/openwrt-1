# ===========================================================================
# OpenWrt configuration targets

.PHONY: clean all $(PHONY)
all: conf mconf
clean:
	rm -f *.o lxdialog/*.o *.moc $(clean-files) conf mconf qconf

# ===========================================================================
# Variables needed by the upstream Makefile

HOSTCC := $(CC)
HOSTCFLAGS:= $(CFLAGS)
CFLAGS += $(HOST_EXTRACFLAGS)
CXXFLAGS += $(HOST_EXTRACFLAGS)

kecho:=true

CONFIG_SHELL:=$(SHELL)
srctree:=.
src:=.
obj:=.
Q:=$(if $V,,@)
cmd = $(cmd_$(1))
dot-target = $(dir $@).$(notdir $@)

### Stripped down upstream Makefile follows
# ===========================================================================

# lxdialog stuff
check-lxdialog  := $(srctree)/$(src)/lxdialog/check-lxdialog.sh

# Use recursively expanded variables so we do not call gcc unless
# we really need to do so. (Do not call gcc as part of make mrproper)
HOST_EXTRACFLAGS += $(shell $(CONFIG_SHELL) $(check-lxdialog) -ccflags) \
                    -DLOCALE

# ===========================================================================
# Shared Makefile for the various kconfig executables:
# conf:	  Used for defconfig, oldconfig and related targets
# mconf:  Used for the menuconfig target
#         Utilizes the lxdialog package
# qconf:  Used for the xconfig target
#         Based on Qt which needs to be installed to compile it
# object files used by all kconfig flavours

lxdialog := lxdialog/checklist.o lxdialog/util.o lxdialog/inputbox.o
lxdialog += lxdialog/textbox.o lxdialog/yesno.o lxdialog/menubox.o

conf-objs	:= conf.o  zconf.tab.o
mconf-objs     := mconf.o zconf.tab.o $(lxdialog)
qconf-cxxobjs	:= qconf.o
qconf-objs	:= zconf.tab.o

hostprogs-y := conf mconf qconf

clean-files	:= qconf.moc .tmp_qtcheck
clean-files	+= zconf.tab.c zconf.lex.c zconf.hash.c

# Check that we have the required ncurses stuff installed for lxdialog (menuconfig)
PHONY += $(obj)/dochecklxdialog
$(addprefix $(obj)/,$(lxdialog)): $(obj)/dochecklxdialog
$(obj)/dochecklxdialog:
	$(Q)$(CONFIG_SHELL) $(check-lxdialog) -check $(HOSTCC) $(HOST_EXTRACFLAGS) $(HOSTLOADLIBES_mconf)

always := dochecklxdialog

HOST_EXTRACFLAGS += -DKBUILD_NO_NLS

# generated files seem to need this to find local include files
HOSTCFLAGS_zconf.lex.o	:= -I$(src)
HOSTCFLAGS_zconf.tab.o	:= -I$(src)

LEX_PREFIX_zconf	:= zconf
YACC_PREFIX_zconf	:= zconf

HOSTLOADLIBES_qconf	= $(KC_QT_LIBS)
HOSTCXXFLAGS_qconf.o	= $(KC_QT_CFLAGS)

HOSTLOADLIBES_mconf   = $(shell $(CONFIG_SHELL) $(check-lxdialog) -ldflags $(HOSTCC))

$(obj)/qconf.o: $(obj)/.tmp_qtcheck

# openwrt: target is qconf, not xconfig
#ifeq ($(MAKECMDGOALS)),xconfig)
ifeq ($(filter qconf,$(MAKECMDGOALS)),qconf)
$(obj)/.tmp_qtcheck: $(src)/Makefile
-include $(obj)/.tmp_qtcheck

# Qt needs some extra effort...
$(obj)/.tmp_qtcheck:
	@set -e; $(kecho) "  CHECK   qt"; \
	if pkg-config --exists Qt5Core; then \
	    cflags="-std=c++11 -fPIC `pkg-config --cflags Qt5Core Qt5Gui Qt5Widgets`"; \
	    libs=`pkg-config --libs Qt5Core Qt5Gui Qt5Widgets`; \
	    moc=`pkg-config --variable=host_bins Qt5Core`/moc; \
	elif pkg-config --exists QtCore; then \
	    cflags=`pkg-config --cflags QtCore QtGui`; \
	    libs=`pkg-config --libs QtCore QtGui`; \
	    moc=`pkg-config --variable=moc_location QtCore`; \
	else \
	    echo >&2 "*"; \
	    echo >&2 "* Could not find Qt via pkg-config."; \
	    echo >&2 "* Please install either Qt 4.8 or 5.x. and make sure it's in PKG_CONFIG_PATH"; \
	    echo >&2 "*"; \
	    exit 1; \
	fi; \
	echo "KC_QT_CFLAGS=$$cflags" > $@; \
	echo "KC_QT_LIBS=$$libs" >> $@; \
	echo "KC_QT_MOC=$$moc" >> $@
endif

$(obj)/zconf.tab.o: $(obj)/zconf.lex.c $(obj)/zconf.hash.c

$(obj)/qconf.o: $(obj)/qconf.moc

quiet_cmd_moc = MOC     $@
      cmd_moc = $(KC_QT_MOC) -i $< -o $@

$(obj)/%.moc: $(src)/%.h $(obj)/.tmp_qtcheck
	$(call cmd,moc)

# ===========================================================================
# OpenWrt rules and final adjustments that need to be made after reading the
# full upstream Makefile

clean-files	+= zconf.tab.h

FORCE:

%.tab.c %.tab.h: %.y
	bison -l -d -b $* -p $(notdir $*) $<

%.lex.c: %.l
	flex -L -P$(notdir $*) -o$@ $<

%.hash.c: %.gperf
	gperf < $< > $@

$(foreach f, mconf.o $(lxdialog), \
  $(eval $f: CFLAGS+=$$(HOSTCFLAGS_$f)))

$(obj)/zconf.lex.o: CFLAGS += $(HOSTCFLAGS_zconf.lex.o)
$(obj)/zconf.tab.o: CFLAGS += $(HOSTCFLAGS_zconf.tab.o)
$(obj)/qconf.o: CXXFLAGS+=$(HOSTCXXFLAGS_qconf.o)

conf: $(conf-objs)

# The *conf-cfg file is used (then filtered out) as the first prerequisite to
# avoid sourcing it before the script is built, when trying to compute CFLAGS
# for the actual first prerequisite.  This avoids errors like:
# '/bin/sh: ./mconf-cfg: No such file or directory'
mconf: $(mconf-objs)
	$(CC) -o $@ $(filter-out mconf-cfg,$^) $(HOSTLOADLIBES_mconf)

qconf: $(qconf-cxxobjs) $(qconf-objs)
	$(CXX) -o $@ $(filter-out qconf-cfg,$^) $(HOSTLOADLIBES_qconf)
