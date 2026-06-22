# Kbuild for the out-of-tree tm16xx build, tailored for ophub kernels.
#
# Why this exists instead of jefflessard's repo Makefile: the ophub kernel config
# enables the other auxdisplay drivers (CONFIG_CHARLCD=m, etc.), so the upstream
# repo Makefile tries to build charlcd.o and friends from sources that aren't in
# an out-of-tree checkout. This builds only the linedisp + tm16xx objects.
#
# The keypad sub-module is intentionally omitted (front panels we target have no
# buttons; including it triggers a tm16xx_set_key redefinition on this kernel).

ccflags-y += -DCONFIG_TM16XX -DCONFIG_TM16XX_I2C -DCONFIG_TM16XX_SPI
ccflags-y += -include $(src)/tm16xx_compat.h -I$(src)/include

obj-m += line-display.o
obj-m += tm16xx.o
tm16xx-y := tm16xx_core.o
obj-m += tm16xx_spi.o
obj-m += tm16xx_i2c.o
