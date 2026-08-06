# Copyright © 2017 Amdocs, Bell Canada
#
# Copyright (c) 2023 Abhijit Gadgil
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

PARENT_CHART := nearrtric
COMMON_CHARTS = nearrt-ric-common
ROOT_DIR := $(shell dirname $(realpath $(lastword $(MAKEFILE_LIST))))
OUTPUT_DIR := $(ROOT_DIR)/dist
PACKAGE_DIR := $(OUTPUT_DIR)/packages
SECRET_DIR := $(OUTPUT_DIR)/secrets
HELM_BIN := helm

HELM_VER := $(shell $(HELM_BIN) version --template "{{.Version}}" 2>/dev/null)

ifneq ($(SKIP_LINT),TRUE)
	HELM_LINT_CMD := $(HELM_BIN) lint
else
	HELM_LINT_CMD := echo "Skipping linting of"
endif

EXCLUDES := config test dist helm $(COMMON_CHARTS) $(PARENT_CHART)
HELM_CHARTS := $(filter-out $(EXCLUDES), $(sort $(patsubst %/.,%,$(wildcard */.)))) $(PARENT_CHART)
HELM_CHARTS_CLEAN := $(sort $(patsubst %,clean-%, $(HELM_CHARTS)))

.PHONY: $(COMMON_CHARTS) $(HELM_CHARTS) helm-repo-update

all: print_helm_bin $(COMMON_CHARTS) helm-repo-update $(HELM_CHARTS)

$(HELM_CHARTS):
	@echo "\n[$@]"
	@make package-$@

$(COMMON_CHARTS):
	@echo "\n[$@]"
	@make package-$@

$(HELM_CHARTS_CLEAN):
	@echo "\n[clean $(patsubst clean-%,%,$@)]"
	@rm -f $(patsubst clean-%,%,$@)/Chart.lock;
	@rm -rf $(patsubst clean-%,%,$@)/charts;

print_helm_bin:
	$(info Using Helm binary ${HELM_BIN} which is helm version ${HELM_VER})

make-%:
	@if [ -f $*/Makefile ]; then make -C $*; fi

dep-%: make-%
	@echo $(HELM_BIN) dep up $*
	@if [ -f $*/Chart.yaml ]; then $(HELM_BIN) dep up $*; fi

lint-%: dep-%
	@if [ -f $*/Chart.yaml ]; then $(HELM_LINT_CMD) $*; fi

package-%: lint-%
	@mkdir -p $(PACKAGE_DIR)
	@if [ -f $*/Chart.yaml ]; then $(HELM_BIN) package -d $(PACKAGE_DIR) $*; fi

clean: $(HELM_CHARTS_CLEAN)
	@rm -f */Chart.lock
	@find . -type f -name '*.tgz' -delete
	@rm -rf $(PACKAGE_DIR)/*

distclean: clean
	@rm -rf $(OUTPUT_DIR)

helm-repo-update:
	@$(HELM_BIN) repo update 2>/dev/null || true

%:
	@:
