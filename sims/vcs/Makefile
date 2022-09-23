#########################################################################################
# vcs makefile
#########################################################################################

#########################################################################################
# general path variables
#########################################################################################
base_dir=$(abspath ../..)
sim_dir=$(abspath .)

#########################################################################################
# include shared variables
#########################################################################################
include $(base_dir)/variables.mk

#########################################################################################
# name of simulator (used to generate *.f arguments file)
#########################################################################################
sim_name = vcs

#########################################################################################
# vcs simulator types and rules
#########################################################################################
sim_prefix = simv
sim = $(sim_dir)/$(sim_prefix)-$(MODEL_PACKAGE)-$(CONFIG)
sim_debug = $(sim_dir)/$(sim_prefix)-$(MODEL_PACKAGE)-$(CONFIG)-debug

include $(base_dir)/vcs.mk

.PHONY: default debug
default: $(sim)
debug: $(sim_debug)


#########################################################################################
# simulaton requirements
#########################################################################################
SIM_FILE_REQS += \
	$(ROCKETCHIP_RSRCS_DIR)/vsrc/TestDriver.v \
	$(TESTCHIP_RSRCS_DIR)/testchipip/csrc/SimSerial.cc \
	$(TESTCHIP_RSRCS_DIR)/testchipip/vsrc/SimSerial.v \
	$(TESTCHIP_RSRCS_DIR)/testchipip/csrc/testchip_tsi.cc \
	$(TESTCHIP_RSRCS_DIR)/testchipip/csrc/testchip_tsi.h \
	$(TESTCHIP_RSRCS_DIR)/testchipip/csrc/SimDRAM.cc \
	$(TESTCHIP_RSRCS_DIR)/testchipip/vsrc/SimDRAM.v \
	$(TESTCHIP_RSRCS_DIR)/testchipip/csrc/mm.h \
	$(TESTCHIP_RSRCS_DIR)/testchipip/csrc/mm.cc \
	$(TESTCHIP_RSRCS_DIR)/testchipip/csrc/mm_dramsim2.h \
	$(TESTCHIP_RSRCS_DIR)/testchipip/csrc/mm_dramsim2.cc \
	$(TESTCHIP_RSRCS_DIR)/testchipip/csrc/SimUART.cc \
	$(TESTCHIP_RSRCS_DIR)/testchipip/vsrc/SimUART.v \
	$(TESTCHIP_RSRCS_DIR)/testchipip/csrc/uart.h \
	$(TESTCHIP_RSRCS_DIR)/testchipip/csrc/uart.cc \
	$(ROCKETCHIP_RSRCS_DIR)/csrc/SimDTM.cc \
	$(ROCKETCHIP_RSRCS_DIR)/csrc/SimJTAG.cc \
	$(ROCKETCHIP_RSRCS_DIR)/vsrc/SimJTAG.v \
	$(ROCKETCHIP_RSRCS_DIR)/csrc/remote_bitbang.h \
	$(ROCKETCHIP_RSRCS_DIR)/csrc/remote_bitbang.cc \
	$(ROCKETCHIP_RSRCS_DIR)/vsrc/plusarg_reader.v \
	$(IOCELL_RSRCS_DIR)/vsrc/IOCell.v \
	$(ROCKETCHIP_RSRCS_DIR)/vsrc/EICG_wrapper.v \
	$(CHIPYARD_RSRCS_DIR)/vsrc/ClockDividerN.sv


# copy files but ignore *.h files in *.f since vcs has +incdir+$(build_dir)
ifeq ($(CIRCT),1)
$(sim_files): $(SIM_FILE_REQS) | $(build_dir)
	cp -f $^ $(build_dir)
	$(foreach file,\
		$^,\
		$(if $(filter %.h,$(file)),\
			,\
			echo "$(addprefix $(build_dir)/, $(notdir $(file)))" >> $@;))
endif

#########################################################################################
# import other necessary rules and variables
#########################################################################################
include $(base_dir)/common.mk

#########################################################################################
# vcs binary and arguments
#########################################################################################
VCS = vcs -full64

VCS_OPTS = $(VCS_CC_OPTS) $(VCS_NONCC_OPTS) $(PREPROC_DEFINES)

#########################################################################################
# vcs build paths
#########################################################################################
model_dir = $(build_dir)/$(long_name)
model_dir_debug = $(build_dir)/$(long_name).debug

#########################################################################################
# vcs simulator rules
#########################################################################################
$(sim): $(sim_vsrcs) $(sim_common_files) $(dramsim_lib) $(EXTRA_SIM_REQS)
	rm -rf $(model_dir)
	$(VCS) $(VCS_OPTS) $(EXTRA_SIM_SOURCES) -o $@ -Mdir=$(model_dir)

$(sim_debug): $(sim_vsrcs) $(sim_common_files) $(dramsim_lib) $(EXTRA_SIM_REQS)
	rm -rf $(model_dir_debug)
	$(VCS) $(VCS_OPTS) $(EXTRA_SIM_SOURCES) -o $@ -Mdir=$(model_dir_debug)  \
	+define+DEBUG

#########################################################################################
# create a vcs vpd rule
#########################################################################################
.PRECIOUS: $(output_dir)/%.vpd %.vpd
$(output_dir)/%.vpd: $(output_dir)/% $(sim_debug)
	(set -o pipefail && $(sim_debug) $(PERMISSIVE_ON) $(SIM_FLAGS) $(EXTRA_SIM_FLAGS) $(VERBOSE_FLAGS) +vcdplusfile=$@ $(PERMISSIVE_OFF) $< </dev/null 2> >(spike-dasm > $<.out) | tee $<.log)

$(output_dir)/none.vpd: $(sim_debug)
	mkdir -p $(output_dir)
	(set -o pipefail && $(sim_debug) $(PERMISSIVE_ON) $(SIM_FLAGS) $(EXTRA_SIM_FLAGS) $(VERBOSE_FLAGS) +vcdplusfile=$@ $(PERMISSIVE_OFF) none </dev/null 2> >(spike-dasm > $(output_dir)/none.out) | tee $(output_dir)/none.log)

.PRECIOUS: $(output_dir)/%.fsdb %.fsdb
$(output_dir)/%.fsdb: $(output_dir)/% $(sim_debug)
	(set -o pipefail && $(sim_debug) $(PERMISSIVE_ON) $(SIM_FLAGS) $(EXTRA_SIM_FLAGS) $(VERBOSE_FLAGS) +fsdbfile=$@ $(PERMISSIVE_OFF) $< </dev/null 2> >(spike-dasm > $<.out) | tee $<.log)

$(output_dir)/none.fsdb: $(sim_debug)
	mkdir -p $(output_dir)
	(set -o pipefail && $(sim_debug) $(PERMISSIVE_ON) $(SIM_FLAGS) $(EXTRA_SIM_FLAGS) $(VERBOSE_FLAGS) +fsdbfile=$@ $(PERMISSIVE_OFF) none </dev/null 2> >(spike-dasm > $(output_dir)/none.out) | tee $(output_dir)/none.log)

#########################################################################################
# general cleanup rules
#########################################################################################
.PHONY: clean clean-sim clean-sim-debug
clean:
	rm -rf $(gen_dir) $(sim_prefix)-* ucli.key

clean-sim:
	rm -rf $(model_dir) $(build_dir)/vc_hdrs.h $(sim) $(sim).daidir ucli.key

clean-sim-debug:
	rm -rf $(model_dir_debug) $(build_dir)/vc_hdrs.h $(sim_debug) $(sim_debug).daidir ucli.key

#########################################################################################
# Launch verdi
#########################################################################################
.PHONY: verdi
verdi: $(sim_debug)
	verdi -elab $(sim_debug).daidir/kdb.elab++ &