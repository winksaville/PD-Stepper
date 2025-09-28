# Makefile for Arduino CLI projects
#
# NOTE on .DELETE_ON_ERROR:
# If compile fails partway through, make deletes the target .bin so a half-written file
# doesn’t appear "up-to-date". This won’t catch: PHONY targets, tools that write/move elsewhere,
# or silent corruption (exit 0). For release/final builds, consider: make clean

# Quiet recursive make "Entering/Leaving directory" lines
MAKEFLAGS += --no-print-directory

JQ := $(shell command -v jq 2>/dev/null)
ifeq ($(JQ),)
$(error "jq not found, please install jq")
endif

# ----- user settings -----
CONFIG := ./arduino-cli.yaml
BUILD  := build
BAUD   ?= 115200
FQBN   ?= esp32:esp32:esp32s3
DTO    ?= 100ms
LFRMT  ?=                # pass `LFRMT=--json` to get JSON; `make list LFRMT=--json`
PORT   ?=                # pass `PORT=/dev/ttyXXX`; e.g. `make upload S=Foo PORT=/dev/ttyUSB0`
CLI_V  ?=                # pass `CLI_V=-v`; e.g. `make compile S=Foo CLI_V=-v`

CLI := arduino-cli $(CLI_V) --config-file "$(CONFIG)"

# --- sketch selection: allow short var S or long SKETCH; strip trailing slash ---
SK      := $(strip $(or $(S),$(SKETCH)))
SKETCH  := $(patsubst %/,%,$(SK))

.PHONY: help init uninit list l compile upload monitor clean c u m cl
.DEFAULT_GOAL := help

help:
	@echo "Targets:"; \
	awk 'BEGIN{FS=":.*## "} \
	  /^[A-Za-z0-9_.-]+:.*## / && $$2 != "alias"{order[++n]=$$1;desc[$$1]=$$2;next} \
	  /^[A-Za-z0-9_.-]+:.*##alias/ {split($$0,p,":");al=p[1];sub(":","",al);sub(/^[ \t]*/,"",p[2]);split(p[2],deps," ");tgt=deps[1];alias[tgt]=al;next} \
	  END{for(i=1;i<=n;i++){name=order[i];if(name in alias)printf "  %-14s %s\n",name" ("alias[name]")",desc[name];else printf "  %-14s %s\n",name,desc[name]} \
	      for (a in alias) if (!(a in desc)) printf "  [warn] alias for missing target: %s -> %s\n", alias[a], a > "/dev/stderr"}' $(MAKEFILE_LIST)
	@echo "";
	@echo "Usage:";
	@echo "  make <target> SKETCH=<SketchDir>";
	@echo "  make <target> S=<SketchDir>";
	@echo "Examples:";
	@echo "  make compile  SKETCH=PD_Stepper_Web_Server";
	@echo "  make upload   S=Basic_Functionality_Test PORT=/dev/ttyUSB0";
	@echo "  make monitor  S=PD_Stepper_Web_Server BAUD=921600";
	@echo "";
	@echo "Notes:";
	@echo "  - <SketchDir> must be a subdir containing <SketchDir>/<SketchDir>.ino";
	@echo "  - Trailing '/' after the sketch name is accepted"


init: ## One-time: init config, install core + libs
	$(CLI) config init
	$(CLI) config set directories.data shared/data
	$(CLI) config set directories.user shared
	$(CLI) config set directories.downloads shared/downloads
#	$(CLI) config set network.connection_timeout 600s
	$(CLI) config add board_manager.additional_urls https://espressif.github.io/arduino-esp32/package_esp32_index.json
	$(CLI) core update-index
	$(CLI) core install "esp32:esp32@2.0.14"	# Version with mbedtls_md5_xxx_ret routines PD_Stepper_Web_Server compiles OK may fail to upload
#	$(CLI) core install "esp32:esp32"		# New version with mbedtls_md5_xxx routines PD_Stepper_Web_Server won't compile
	$(CLI) lib install "ESPAsyncWebServer"
	$(CLI) lib install "TMC2209"

uninit: ## Remove all files/dirs created by running `make init`
	rm -rf arduino-cli.yaml build shared

list: ## List connected boards
	$(CLI) board list $(LFRMT) --discovery-timeout "$(DTO)"
l: list ##alias

# delete half-baked binary if compile fails
.DELETE_ON_ERROR: $(BUILD)/$(SKETCH).ino.bin

# Build product from ./<SKETCH>/<SKETCH>.ino
$(BUILD)/$(SKETCH).ino.bin: ./$(SKETCH)/$(SKETCH).ino
	@test -n "$(SKETCH)" || { echo "Error: set S=<SketchDir> (or SKETCH=)"; exit 1; }
	@printf "Compiling sketch '%s' for %s...\n" "$(SKETCH)" "$(FQBN)"
	@$(CLI) compile -b "$(FQBN)" --build-path "$(BUILD)" "./$(SKETCH)"

compile: ## Compile (requires S or SKETCH)
	@test -n "$(SKETCH)" || { echo "Error: set S=<SketchDir> (or SKETCH=)"; exit 1; }
	@$(MAKE) $(BUILD)/$(SKETCH).ino.bin
c:  compile  ##alias

# Helper: resolve port (use PORT if set, else auto-detect), then run $(1)
define GET_PORT
	@P="$$( [ -n "$(PORT)" ] && printf "%s" "$(PORT)" \
	      || $(CLI) board list --discovery-timeout "$(DTO)" --json | jq -r '.detected_ports[0]?.port.address // empty' )"; \
	test -n "$$P" || { echo "Error: no serial port detected. Set PORT=/dev/ttyXXX"; exit 1; }; \
	$(1)
endef

upload: compile ## Upload the built sketch
	$(call GET_PORT,$(CLI) upload -p "$$P" -b "$(FQBN)" --input-dir "$(BUILD)")
u:  upload   ##alias

monitor: ## Open serial monitor
	$(call GET_PORT,$(CLI) monitor -p "$$P" -c baudrate="$(BAUD)")
m:  monitor  ##alias

clean: ## Remove build artifacts
	rm -rf -- "$(BUILD)"
cl: clean    ##alias
