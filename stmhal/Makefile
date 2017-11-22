# 选择要编译的硬件平台: 如果在命令行中没有给予参数
# 则默认选择 EMOROBOTV10.

#?= 如果没有赋值，则赋值
BOARD ?= EMOROBOTV10
#调试信息输出$()输出信息
$(info  $(wildcard boards/$(BOARD)/.) )
ifeq ($(wildcard boards/$(BOARD)/.),)
$(error 要编译的板子不存在)
endif

# 如果编译输出路径没有给出即以板子名字命名输出
BUILD ?= build-$(BOARD)
$(info  $(BUILD)) 

include ../py/mkenv.mk
#包含IO，设备接口的定义
-include mpconfigport.mk

#mpconfigboard.mk中定义了以下几个项
#MCU_SERIES = f4
#CMSIS_MCU = STM32F405xx
#AF_FILE = boards/stm32f405_af.csv
#LD_FILE = boards/stm32f405.ld
include boards/$(BOARD)/mpconfigboard.mk

# qstr 的定义 (必须在包含 py.mk文件之前)
QSTR_DEFS = qstrdefsport.h $(BUILD)/pins_qstr.h $(BUILD)/modstm_qstr.h
$(info QSTR_DEFS=$(QSTR_DEFS))

# directory containing scripts to be frozen as bytecode
FROZEN_MPY_DIR ?= modules
$(info  FROZEN_MPY_DIR=$(FROZEN_MPY_DIR))

# 包含py内核的Make文件
include $(TOP)/py/py.mk
$(info  TOP=$(TOP)/py/py.mk) 

#平台路径
LD_DIR=boards
#CMSIS内核路径
CMSIS_DIR=cmsis

#硬件抽象接口路径
HAL_DIR=hal/$(MCU_SERIES)
$(info  HAL_DIR=hal/$(MCU_SERIES)) 

#USB设备路径
USBDEV_DIR=usbdev
#USBHOST_DIR=usbhost

#文件系统路径
FATFS_DIR=lib/oofatfs

#DFU 的PYthon支持，以下是对dfu的配置
DFU=$(TOP)/tools/dfu.py
# may need to prefix dfu-util with sudo
#选择支持DFU模式
USE_PYDFU ?= 1
PYDFU ?= $(TOP)/tools/pydfu.py
DFU_UTIL ?= dfu-util
DEVICE=0483:df11
STFLASH ?= st-flash
OPENOCD ?= openocd
OPENOCD_CONFIG ?= boards/openocd_stm32f4.cfg

#ARM交叉编译工具arm-none-eabi
CROSS_COMPILE = arm-none-eabi-

#添加头文件目录
INC += -I.
INC += -I$(TOP)
INC += -I$(BUILD)
INC += -I$(TOP)/lib/cmsis/inc
INC += -I$(CMSIS_DIR)/
INC += -I$(HAL_DIR)/inc
INC += -I$(USBDEV_DIR)/core/inc -I$(USBDEV_DIR)/class/inc
#INC += -I$(USBHOST_DIR)
$(info  INC=$(INC)) 

#以下是编译设置
# Basic Cortex-M flags
CFLAGS_CORTEX_M = -mthumb

# Select hardware floating-point support
ifeq ($(CMSIS_MCU),$(filter $(CMSIS_MCU),STM32F767xx STM32F769xx))
CFLAGS_CORTEX_M += -mfpu=fpv5-d16 -mfloat-abi=hard
else
CFLAGS_CORTEX_M += -mfpu=fpv4-sp-d16 -mfloat-abi=hard
endif
$(info  CFLAGS_CORTEX_M=$(CFLAGS_CORTEX_M))


# Options for particular MCU series
#选择特定的MCU系列
CFLAGS_MCU_f4 = $(CFLAGS_CORTEX_M) -mtune=cortex-m4 -mcpu=cortex-m4 -DMCU_SERIES_F4
CFLAGS_MCU_f7 = $(CFLAGS_CORTEX_M) -mtune=cortex-m7 -mcpu=cortex-m7 -DMCU_SERIES_F7
CFLAGS_MCU_l4 = $(CFLAGS_CORTEX_M) -mtune=cortex-m4 -mcpu=cortex-m4 -DMCU_SERIES_L4

#编译设置
#-Wall：选项意思是编译后显示所有警告。
#-Wpointer-arith：对函数指针或者void *类型的指针进行算术操作时给出警告。也很有用。 -Wall 并不会打开此项。
#-Werror：把警告当作错误。出现任何警告就放弃编译。
#-std=gnu99：使用 ISO C99 再加上 GNU 的一些扩展。
#-nostdlib：不连接系统标准启动文件和标准库文件，只把指定的文件传递给连接器
#CFLAGS_MOD 来自/py/mkenv.mk文件，设置文件系统路径
#CFLAGS_EXTRA：这里是空
CFLAGS = $(INC) -Wall -Wpointer-arith -Werror -std=gnu99 -nostdlib $(CFLAGS_MOD) $(CFLAGS_EXTRA)
CFLAGS += -D$(CMSIS_MCU)
CFLAGS += $(CFLAGS_MCU_$(MCU_SERIES))
CFLAGS += $(COPT)
CFLAGS += -Iboards/$(BOARD)
CFLAGS += -DSTM32_HAL_H='<stm32$(MCU_SERIES)xx_hal.h>'
$(info  CFLAGS=$(CFLAGS))

ifeq ($(MICROPY_FLOAT_IMPL),double)
CFLAGS += -DMICROPY_FLOAT_IMPL=MICROPY_FLOAT_IMPL_DOUBLE
else
CFLAGS += -DMICROPY_FLOAT_IMPL=MICROPY_FLOAT_IMPL_FLOAT
CFLAGS += -fsingle-precision-constant -Wdouble-promotion
endif
$(info  CFLAGS=$(CFLAGS))

# Remove uncalled code from the final image.
CFLAGS += -fdata-sections -ffunction-sections
LDFLAGS += --gc-sections

# Debugging/Optimization
ifeq ($(DEBUG), 1)
CFLAGS += -g -DPENDSV_DEBUG
COPT = -O0
else
COPT += -Os -DNDEBUG
endif

#Lib源码文件加载
SRC_LIB = $(addprefix lib/,\
	libc/string0.c \
	oofatfs/ff.c \
	oofatfs/option/unicode.c \
	mp-readline/readline.c \
	netutils/netutils.c \
	timeutils/timeutils.c \
	utils/pyexec.c \
	utils/interrupt_char.c \
	utils/sys_stdio_mphal.c \
	)
$(info  SRC_LIB=$(SRC_LIB))


#lib/libm源码加载
#本文件中MICROPY_FLOAT_IMPL为空
ifeq ($(MICROPY_FLOAT_IMPL),double)
SRC_LIBM = $(addprefix lib/libm_dbl/,\
	__cos.c \
	__expo2.c \
	__fpclassify.c \
	__rem_pio2.c \
	__rem_pio2_large.c \
	__signbit.c \
	__sin.c \
	__tan.c \
	acos.c \
	acosh.c \
	asin.c \
	asinh.c \
	atan.c \
	atan2.c \
	atanh.c \
	ceil.c \
	cos.c \
	cosh.c \
	erf.c \
	exp.c \
	expm1.c \
	floor.c \
	fmod.c \
	frexp.c \
	ldexp.c \
	lgamma.c \
	log.c \
	log10.c \
	log1p.c \
	modf.c \
	nearbyint.c \
	pow.c \
	rint.c \
	scalbn.c \
	sin.c \
	sinh.c \
	sqrt.c \
	tan.c \
	tanh.c \
	tgamma.c \
	trunc.c \
	)
else
SRC_LIBM = $(addprefix lib/libm/,\
	math.c \
	thumb_vfp_sqrtf.c \
	acoshf.c \
	asinfacosf.c \
	asinhf.c \
	atan2f.c \
	atanf.c \
	atanhf.c \
	ef_rem_pio2.c \
	erf_lgamma.c \
	fmodf.c \
	kf_cos.c \
	kf_rem_pio2.c \
	kf_sin.c \
	kf_tan.c \
	log1pf.c \
	nearbyintf.c \
	sf_cos.c \
	sf_erf.c \
	sf_frexp.c \
	sf_ldexp.c \
	sf_modf.c \
	sf_sin.c \
	sf_tan.c \
	wf_lgamma.c \
	wf_tgamma.c \
	)
endif
$(info  SRC_LIBM=$(SRC_LIBM))

#额外模块源码
EXTMOD_SRC_C = $(addprefix extmod/,\
	modonewire.c \
        )
$(info  EXTMOD_SRC_C=$(EXTMOD_SRC_C))

#SPIfalsh接口源码
DRIVERS_SRC_C = $(addprefix drivers/,\
	memory/spiflash.c \
	)
$(info  DRIVERS_SRC_C=$(DRIVERS_SRC_C))

#主要的C源码
SRC_C = \
	main.c \
	system_stm32.c \
	stm32_it.c \
	usbd_conf.c \
	usbd_desc.c \
	usbd_cdc_interface.c \
	usbd_hid_interface.c \
	usbd_msc_storage.c \
	mphalport.c \
	mpthreadport.c \
	irq.c \
	pendsv.c \
	systick.c  \
	pybthread.c \
	timer.c \
	led.c \
	pin.c \
	pin_defs_stmhal.c \
	pin_named_pins.c \
	bufhelper.c \
	dma.c \
	i2c.c \
	spi.c \
	uart.c \
	can.c \
	usb.c \
	wdt.c \
	gccollect.c \
	help.c \
	machine_i2c.c \
	modmachine.c \
	modpyb.c \
	modstm.c \
	moduos.c \
	modutime.c \
	modusocket.c \
	modnetwork.c \
	extint.c \
	usrsw.c \
	rng.c \
	rtc.c \
	flash.c \
	storage.c \
	sdcard.c \
	fatfs_port.c \
	lcd.c \
	accel.c \
	servo.c \
	dac.c \
	adc.c \
	$(wildcard boards/$(BOARD)/*.c)
	
$(info  SRC_C=$(SRC_C))	
#调试目录路径
#all: 
#	ls $(SRC_C)

	
SRC_O = \
	startup_stm32.o \
	gchelper.o \	
	
SRC_HAL = $(addprefix $(HAL_DIR)/src/stm32$(MCU_SERIES)xx_,\
	hal.c \
	hal_adc.c \
	hal_adc_ex.c \
	hal_can.c \
	hal_cortex.c \
	hal_dac.c \
	hal_dac_ex.c \
	hal_dma.c \
	hal_flash.c \
	hal_flash_ex.c \
	hal_gpio.c \
	hal_i2c.c \
	hal_pcd.c \
	hal_pcd_ex.c \
	hal_pwr.c \
	hal_pwr_ex.c \
	hal_rcc.c \
	hal_rcc_ex.c \
	hal_rng.c \
	hal_rtc.c \
	hal_rtc_ex.c \
	hal_sd.c \
	hal_spi.c \
	hal_tim.c \
	hal_tim_ex.c \
	hal_uart.c \
	ll_sdmmc.c \
	ll_usb.c \
	)
$(info  SRC_HAL=$(SRC_HAL))

SRC_USBDEV = $(addprefix $(USBDEV_DIR)/,\
	core/src/usbd_core.c \
	core/src/usbd_ctlreq.c \
	core/src/usbd_ioreq.c \
	class/src/usbd_cdc_msc_hid.c \
	class/src/usbd_msc_bot.c \
	class/src/usbd_msc_scsi.c \
	class/src/usbd_msc_data.c \
	)
$(info  SRC_USBDEV=$(SRC_USBDEV))	

#for wiznet5k module 这里是没有
ifeq ($(MICROPY_PY_WIZNET5K),1)
WIZNET5K_DIR=drivers/wiznet5k
INC += -I$(TOP)/$(WIZNET5K_DIR)
CFLAGS_MOD += -DMICROPY_PY_WIZNET5K=1
SRC_MOD += modnwwiznet5k.c
SRC_MOD += $(addprefix $(WIZNET5K_DIR)/,\
	ethernet/w5200/w5200.c \
	ethernet/wizchip_conf.c \
	ethernet/socket.c \
	internet/dns/dns.c \
	)
endif
$(info  SRC_MOD=$(SRC_MOD))

# for CC3000 module  这里是没有
ifeq ($(MICROPY_PY_CC3K),1)
CC3000_DIR=drivers/cc3000
INC += -I$(TOP)/$(CC3000_DIR)/inc
CFLAGS_MOD += -DMICROPY_PY_CC3K=1
SRC_MOD += modnwcc3k.c
SRC_MOD += $(addprefix $(CC3000_DIR)/src/,\
	cc3000_common.c \
	evnt_handler.c \
	hci.c \
	netapp.c \
	nvmem.c \
	security.c \
	socket.c \
	wlan.c \
	ccspi.c \
	inet_ntop.c \
	inet_pton.c \
	patch.c \
	patch_prog.c \
	)
endif	
$(info  SRC_MOD=$(SRC_MOD))



OBJ =
OBJ += $(PY_O)
OBJ += $(addprefix $(BUILD)/, $(SRC_LIB:.c=.o))
OBJ += $(addprefix $(BUILD)/, $(SRC_LIBM:.c=.o))
OBJ += $(addprefix $(BUILD)/, $(EXTMOD_SRC_C:.c=.o))
OBJ += $(addprefix $(BUILD)/, $(DRIVERS_SRC_C:.c=.o))
OBJ += $(addprefix $(BUILD)/, $(SRC_C:.c=.o))
OBJ += $(addprefix $(BUILD)/, $(SRC_O))
OBJ += $(addprefix $(BUILD)/, $(SRC_HAL:.c=.o))
OBJ += $(addprefix $(BUILD)/, $(SRC_USBDEV:.c=.o))
OBJ += $(addprefix $(BUILD)/, $(SRC_MOD:.c=.o))
OBJ += $(BUILD)/pins_$(BOARD).o
$(info  OBJ=$(OBJ))

# We put several files into the first 16K section with the ISRs.
# If we compile these using -O0 then it won't fit. So if you really want these
# to be compiled with -O0, then edit boards/common.ld (in the .isr_vector section)
# and comment out the following lines.
$(info  PY_BUILD=看看这里之前是否已经赋值，$(PY_BUILD))
$(BUILD)/$(FATFS_DIR)/ff.o: COPT += -Os
$(filter $(PY_BUILD)/../extmod/vfs_fat_%.o, $(PY_O)): COPT += -Os
$(PY_BUILD)/formatfloat.o: COPT += -Os
$(PY_BUILD)/parsenum.o: COPT += -Os
$(PY_BUILD)/mpprint.o: COPT += -Os
$(info  PY_BUILD=$(PY_BUILD))


#生成dfu和hex
all: $(BUILD)/firmware.dfu $(BUILD)/firmware.hex


ifneq ($(FROZEN_DIR),)
# To use frozen source modules, put your .py files in a subdirectory (eg scripts/)
# and then invoke make with FROZEN_DIR=scripts (be sure to build from scratch).
CFLAGS += -DMICROPY_MODULE_FROZEN_STR
endif

ifneq ($(FROZEN_MPY_DIR),)
# To use frozen bytecode, put your .py files in a subdirectory (eg frozen/) and
# then invoke make with FROZEN_MPY_DIR=frozen (be sure to build from scratch).
CFLAGS += -DMICROPY_QSTR_EXTRA_POOL=mp_qstr_frozen_const_pool
CFLAGS += -DMICROPY_MODULE_FROZEN_MPY
endif

.PHONY: deploy

deploy: $(BUILD)/firmware.dfu
	$(ECHO) "Writing $< to the board"
ifeq ($(USE_PYDFU),1)
	$(Q)$(PYTHON) $(PYDFU) -u $<
else
	$(Q)$(DFU_UTIL) -a 0 -d $(DEVICE) -D $<
endif

FLASH_ADDR ?= 0x08000000
TEXT_ADDR ?= 0x08020000

deploy-stlink: $(BUILD)/firmware.dfu
	$(ECHO) "Writing $(BUILD)/firmware0.bin to the board via ST-LINK"
	$(Q)$(STFLASH) write $(BUILD)/firmware0.bin $(FLASH_ADDR)
	$(ECHO) "Writing $(BUILD)/firmware1.bin to the board via ST-LINK"
	$(Q)$(STFLASH) --reset write $(BUILD)/firmware1.bin $(TEXT_ADDR)

deploy-openocd: $(BUILD)/firmware.dfu
	$(ECHO) "Writing $(BUILD)/firmware{0,1}.bin to the board via ST-LINK using OpenOCD"
	$(Q)$(OPENOCD) -f $(OPENOCD_CONFIG) -c "stm_flash $(BUILD)/firmware0.bin $(FLASH_ADDR) $(BUILD)/firmware1.bin $(TEXT_ADDR)"

$(BUILD)/firmware.dfu: $(BUILD)/firmware.elf
	$(ECHO) "Create $@"
	$(Q)$(OBJCOPY) -O binary -j .isr_vector $^ $(BUILD)/firmware0.bin
	$(Q)$(OBJCOPY) -O binary -j .text -j .data $^ $(BUILD)/firmware1.bin
	$(Q)$(PYTHON) $(DFU) -b $(FLASH_ADDR):$(BUILD)/firmware0.bin -b $(TEXT_ADDR):$(BUILD)/firmware1.bin $@

$(BUILD)/firmware.hex: $(BUILD)/firmware.elf
	$(ECHO) "Create $@"
	$(Q)$(OBJCOPY) -O ihex $< $@

$(BUILD)/firmware.elf: $(OBJ)
	$(ECHO) "LINK $@"
	$(Q)$(LD) $(LDFLAGS) -o $@ $^ $(LIBS)
	$(Q)$(SIZE) $@

