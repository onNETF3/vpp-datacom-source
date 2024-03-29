# Copyright (c) 2021 Comcast Cable Communications Management, LLC.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at:
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#
# Build the documentation
#

# Default target
.PHONY: all
all: siphon

# These should be passed in by the root Makefile
WS_ROOT ?= $(CURDIR)/../..
BR ?= $(WS_ROOT)/build-root

# Tag used in github repository path.
# Change this when genearting for a release
VPP_TAG ?= master

REPOSITORY_URL ?= https://github.com/FDio/vpp/blob/$(VPP_TAG)/

# Doxygen configuration and our utility scripts
SCRIPTS_DIR ?= $(WS_ROOT)/docs/_scripts

# docs root directory
DOCS_DIR ?= ${BR}/docs/src

FEATURE_LIST_FILE = ${DOCS_DIR}/aboutvpp/featurelist.md

# Siphoned fragements are processed into here
DOCS_GENERATED_DIR ?= $(DOCS_DIR)/_generated

# Siphoned fragments end up in here
SIPHON_INPUT_DIR ?= $(DOCS_GENERATED_DIR)/fragments

DYNAMIC_RENDER_DIR ?= ${DOCS_GENERATED_DIR}/includes

# Primary source directories
SIPHON_SRC ?= $(WS_ROOT)/src
SIPHON_SRC_DIRECTORIES = \
	$(shell find $(SIPHON_SRC) -name '*.md' -print | xargs dirname \
	| sort | uniq) \
	$(SIPHON_SRC)/vppinfra \
	$(SIPHON_SRC)/svm \
	$(SIPHON_SRC)/vlib \
	$(SIPHON_SRC)/vlibapi \
	$(SIPHON_SRC)/vlibmemory \
	$(SIPHON_SRC)/vnet \
	$(SIPHON_SRC)/vpp \
	$(SIPHON_SRC)/vpp-api \
	$(SIPHON_SRC)/examples

# Input directories and files
SIPHON_INPUT ?= \
	$(wildcard $(WS_ROOT)/*.md) \
	$(wildcard $(SCRIPTS_DIR)/*.md) \
	$(SIPHON_SRC_DIRECTORIES) \
	$(SIPHON_SRC)/plugins \
	extras

# Strip leading workspace path from input names
SIPHON_INPUT := $(subst $(WS_ROOT)/,,$(SIPHON_INPUT))

# Files to exclude, from pre-Doxygen steps, eg because they're
# selectively compiled.
# Examples would be to exclude non-DPDK related sources when
# there's a DPDK equivalent that conflicts.
# These must be left-anchored paths for the regexp below to work.
SIPHON_EXCLUDE ?= \
	$(SIPHON_SRC)/vpp-api/lua

# Generate a regexp for filenames to exclude
SIPHON_EXCLUDE_REGEXP = ($(subst .,\.,$(shell echo '$(strip $(SIPHON_EXCLUDE))' | sed -e 's/ /|/g')))

# Include all the normal source directories in the include file path
SIPHON_INCLUDE_PATH = $(SIPHON_SRC_DIRECTORIES)

# Find API header directories and include them in the header path.
# This is only useful if VPP and plugins are already built; nothing
# here depends on those targets. We don't build documentation for these
# header files, they're just added to the INCLUDE search path for Doxygen.
_vpp_br = $(shell find "$(BR)" -maxdepth 1 -type d \
	'(' -name build-vpp_debug-native -o -name build-vpp-native ')' -print \
	| sed -e 's@^$(WS_ROOT)/*@@' -e 1q)
ifneq ($(strip $(_vpp_br)),)
SIPHON_INCLUDE_PATH += \
	$(_vpp_br)/vlib-api \
	$(_vpp_br)/vpp
# Also include any plugin directories that exist
SIPHON_INCLUDE_PATH += \
	$(shell find $(WS_ROOT)/$(_vpp_br)/plugins -maxdepth 1 -type d | sed -e 's@^$(WS_ROOT)/*@@')
endif

# Discover if we have CPP available
_cpp = $(shell which cpp)
ifneq ($(strip $(_cpp)),)
# Add whatever directories CPP normally includes to the header path
SIPHON_INCLUDE_PATH += $(shell set -e; $(_cpp) -v </dev/null 2>&1 | awk 'f&&/^ /{print $$1} /^\#include/{f=1}')
endif

# All the siphon types we know about
SIPHONS ?= clicmd syscfg

SIPHON_FILES = $(addprefix $(SIPHON_INPUT_DIR)/,$(addsuffix .siphon,$(SIPHONS)))
SIPHON_DOCS = $(addprefix $(DOCS_GENERATED_DIR)/,$(addsuffix .rst,$(SIPHONS)))

BUILT_ON = $(shell date '+%d %B %Y')
VPP_VERSION = $(shell ${WS_ROOT}/src/scripts/version)

.PHONY: featurelist
featurelist:
	@( \
	  cd $(WS_ROOT) && \
	  find . -name FEATURE.yaml | \
	    ./src/scripts/fts.py \
	      --markdown \
	      --repolink $(REPOSITORY_URL) > \
	    $(FEATURE_LIST_FILE) ; \
	)


.PHONY: includes-render
includes-render:
	@mkdir -p "$(DYNAMIC_RENDER_DIR)"
	@python3 $(SCRIPTS_DIR)/includes_renderer.py ${WS_ROOT} ${DYNAMIC_RENDER_DIR}

.PHONY: template-index
template-index:
	@sed -ie "s/__VPP_VERSION__/${VPP_VERSION}/g" ${DOCS_DIR}/index.rst
	@sed -ie "s/__BUILT_ON__/${BUILT_ON}/g" ${DOCS_DIR}/index.rst
	@( \
	  for f in $$(grep -l -R __REPOSITORY_URL__ ${DOCS_DIR} | grep -e '\.rst$$' -e '\.md$$' ) ;\
	  do \
	    if [ ! -z $${f} ]; then \
		echo "TEMPLATING $${f}" ;\
		sed -ie "s@__REPOSITORY_URL__@${REPOSITORY_URL}@g" $${f} ;\
	    fi ;\
	  done ; \
	)

.NOTPARALLEL: $(SIPHON_FILES)
$(SIPHON_FILES): $(SCRIPTS_DIR)/siphon-generate \
		$(addprefix,$(WSROOT),$(SIPHON_INPUT)) \
		$(wildcard $(SCRIPTS_DIR)/siphon/*.py)
	@echo "Validating source tree..."
	@set -e; for input in $(SIPHON_INPUT); do \
		if [ ! -e "$(WS_ROOT)/$$input" ]; then \
			echo "ERROR: Input path '$$input' does not exist." >&2; \
			exit 1; \
		fi; \
	done
	@rm -rf "$(SIPHON_INPUT_DIR)" "$(DOCS_GENERATED_DIR)"
	@mkdir -p "$(SIPHON_INPUT_DIR)" "$(DOCS_GENERATED_DIR)"
	@touch $(SIPHON_INPUT_DIR)/files
	@echo "Collating source file list for siphoning..."
	@for input in $(SIPHON_INPUT); do \
		cd "$(WS_ROOT)"; \
		find "$$input" -type f \
			\( -name '*.[ch]' -or -name '*.dox' \) -print \
			| grep -v -E '^src/examples/' \
			| grep -v -E '^$(SIPHON_EXCLUDE_REGEXP)' \
			>> $(SIPHON_INPUT_DIR)/files; \
	done
	@echo "Generating siphons..."
	@set -e; \
	cd "$(WS_ROOT)"; \
	$(SCRIPTS_DIR)/siphon-generate \
		--output="$(SIPHON_INPUT_DIR)" \
		"@$(SIPHON_INPUT_DIR)/files"

# Evaluate this to build a siphon doc output target for each desired
# output type:
# $(eval $(call siphon-process,file_extension,output_type_name))
define siphon-process
$(DOCS_GENERATED_DIR)/%.$(1): $(SIPHON_INPUT_DIR)/%.siphon \
		$(SCRIPTS_DIR)/siphon-process \
		$(wildcard $(SCRIPTS_DIR)/siphon/*.py) \
		$(wildcard $(SCRIPTS_DIR)/siphon_templates/$(2)/*/*.$(1))
	@echo "Processing siphon for $(2) from $$(notdir $$<)..."
	@set -e; \
	cd "$(WS_ROOT)"; \
	mkdir -p $(DOCS_GENERATED_DIR)/$$(basename $$(notdir $$<)).$(1).dir; \
	$(SCRIPTS_DIR)/siphon-process \
		--type=$$(basename $$(notdir $$<)) \
		--format=$(2) \
		--repolink=$(REPOSITORY_URL)/ \
		--outdir=$(DOCS_GENERATED_DIR)/$$(basename $$(notdir $$<)).$(1).dir \
		--output="$$@" \
		"$$<"
endef

# Process the .siphon source fragments and render them into siphon flavored
# markdown documentation
.DELETE_ON_ERROR: $(SIPHON_DOCS)
$(eval $(call siphon-process,rst,markdown))

# This target can be used just to generate the siphoned things
.PHONY: siphon
siphon: $(SIPHON_DOCS)
	@cp $(DOCS_GENERATED_DIR)/clicmd.rst $(DOCS_DIR)/cli-reference/index.rst
	@cp -r $(DOCS_GENERATED_DIR)/clicmd.rst.dir $(DOCS_DIR)/cli-reference/clis

.PHONY: generate
generate: siphon includes-render template-index featurelist

.PHONY: clean
clean:
	@rm -rf $(BR)/.siphon.dep
	@rm -rf $(SCRIPTS_DIR)/siphon/__pycache__

