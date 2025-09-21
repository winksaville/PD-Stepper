# Makefile for Arduino CLI projects
#
# NOTE on .DELETE_ON_ERROR:
# We scope it to the binary so if the compile rule fails partway through,
# make will delete the *target file* instead of leaving a half-written .bin
# that could appear "up-to-date" later. This won’t catch failures when:
#   - the failing target is PHONY (no file to delete),
#   - the tool writes elsewhere or moves files after failure,
#   - corruption happens but the command still exits 0.
# For release/final builds, consider running:  make clean  first.

JQ := $(shell command -v jq 2>/dev/null)
ifeq ($(JQ),)
$(error "jq not found, please install jq")
endif

# ----- user settings -----
CONFIG := ./arduino-cli.yaml
SKETCH := PD_Stepper_Web_Server.ino
BUILD  := build
BAUD   ?= 115200
FQBN   ?= esp32:esp32:esp32s3
DTO    ?= 100ms
LFRMT  ?=                # pass `LFRMT=--json` to get JSON; `make list LFRMT=--json`
PORT   ?=                # pass `PORT=/dev/ttyXXX` to skip auto detect; `make upload PORT=/dev/ttyUSB0`

CLI_V  ?=                # pass `CLI_V=-v` on command line for verbose cli output; `make xxx CLI_V=-v`
CLI := arduino-cli $(CLI_V) --config-file "$(CONFIG)"

.PHONY: help init compile list upload monitor clean c u m l cl
.DEFAULT_GOAL := help

help:
	@echo "Targets:"; \
	awk 'BEGIN{FS=":.*## "} \
	  /^[A-Za-z0-9_.-]+:.*## / && $$2 != "alias" { order[++n]=$$1; desc[$$1]=$$2; next } \
	  /^[A-Za-z0-9_.-]+:.*##alias/ { split($$0,p,":"); al=p[1]; sub(":","",al); sub(/^[ \t]*/,"",p[2]); split(p[2],deps," "); tgt=deps[1]; alias[tgt]=al } \
	  END{ for(i=1;i<=n;i++){ name=order[i]; \
	    if(name in alias) printf "  %-14s %s\n", name " (" alias[name] ")", desc[name]; \
	    else               printf "  %-14s %s\n", name, desc[name]; \
	  }}' $(MAKEFILE_LIST)

init: ## One-time: init config, install core + libs
	$(CLI) config init
#	$(CLI) config set network.connection_timeout 600s	# Enable if internet is slow and installing core or libraries fail
	$(CLI) config add board_manager.additional_urls https://espressif.github.io/arduino-esp32/package_esp32_index.json 
	$(CLI) config set directories.data ./data
	$(CLI) config set directories.user .
	$(CLI) config set directories.downloads ./downloads
	$(CLI) core update-index
	$(CLI) core install "esp32:esp32@2.0.14" 	# Version with mbedtls_md5_xxx_ret routines
#	$(CLI) core install "esp32:esp32"		# EXPECT `make compile` TO FAIL as mbdtls_md5_xxx_ret routines do not exist. Renaming them, dropping `_ret`, in WebAuthentication.cpp does allow the `make compile` to succeed
#						 	# with WebAuthentication.cpp and we get several errors like:
#						 	# /home/wink/data/prgs/PD-Stepper/Software/PD_Stepper_Web_Server/libraries/ESPAsyncWebServer/src/WebAuthentication.cpp: In function 'bool getMD5(uint8_t*, uint16_t, char*)':
#						 	# /home/wink/data/prgs/PD-Stepper/Software/PD_Stepper_Web_Server/libraries/ESPAsyncWebServer/src/WebAuthentication.cpp:74:3: error: 'mbedtls_md5_starts_ret' was not declared in this scope; did you mean 'mbedtls_md5_starts'?
#						 	#    74 |   mbedtls_md5_starts_ret(&_ctx);
#						 	#       |   ^~~~~~~~~~~~~~~~~~~~~~
#						 	#       |   mbedtls_md5_starts
#						 	#
	$(CLI) lib install "ESPAsyncWebServer"
	$(CLI) lib install "TMC2209"

uninit: ## Remove all files/dirs created by running `make init`
	rm -rf arduino-cli.yaml build data downloads libraries

# delete half-baked binary if compile fails
.DELETE_ON_ERROR: $(BUILD)/$(SKETCH).bin

$(BUILD)/$(SKETCH).bin: $(SKETCH)
	printf "Compiling sketch '%s' for %s...\n" "$(SKETCH)" "$(FQBN)"
	$(CLI) compile -b "$(FQBN)" --build-path "$(BUILD)"

compile: $(BUILD)/$(SKETCH).bin ## Compile if sketch changed
c: compile ##alias

# Helper: resolve port (use PORT if set, else auto-detect), then run $(1)
define GET_PORT
	@P="$$( [ -n "$(PORT)" ] && printf "%s" "$(PORT)" \
	      || $(CLI) board list --discovery-timeout "$(DTO)" --json | jq -r '.detected_ports[0]?.port.address // empty' )"; \
	test -n "$$P" || { echo "Error: no serial port detected. Set PORT=/dev/ttyXXX"; exit 1; }; \
	$(1)
endef

list: ## List connected boards
	$(CLI) board list $(LFRMT) --discovery-timeout "$(DTO)"
l: list ##alias

upload: compile ## Upload the built sketch
	$(call GET_PORT,$(CLI) upload -p "$$P" -b "$(FQBN)" --input-dir "$(BUILD)")
u: upload ##alias

monitor: ## Open serial monitor
	$(call GET_PORT,$(CLI) monitor -p "$$P" -c baudrate="$(BAUD)")
m: monitor ##alias

clean: ## Remove build artifacts
	rm -rf -- "$(BUILD)"
cl: clean ##alias
