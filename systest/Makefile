# Copyright 2016-2017 F5 Networks Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

#
# This file is used by the F5 OpenStack team as an interface to the automated
# test system. It is divided into 4 sections:
# VARIABLES
# INITIALIZATION
# UNITTEST
# MULTIPLEXING
# TESTSBYINFRA

#### VARIABLES SECTION:
#    DEVICES: (See: https://support.f5.com/csp/article/K8986)
#     A list of MAJOR_MINOR_MAINTENANCE, adding a new triple XX.Y.Z causes
#     the system to look up the "current actual" image in the
#     "device_version_maps.sh" file.  For a given XX.Y.Z device the system
#     tests against a single (OpenStack Ready) image listed in that file.  The
#     assumption is that the XX.Y.Z triple specifies a hardware platform, and
#     futher verison info denotes software _for_ that hardware.
#    DEPLOYS are a list of XX.Y.Z triples combined with other variables,
#     currently the only specified variable is the (implicit) "CLOUDTYPE" with
#     the states "overcloud" and "undercloud", therefore elements of DEPLOYS
#     look like the following: '12.1.1-overcloud' '11.5.4-undercloud'.
#
DEVICES := 12.1.1
DEPLOYS := $(foreach v, $(DEVICES), $(v)-overcloud $(v)-undercloud)

# Tell make that targets don't map to files
.PHONY: setup \
	unit_test \
        $(DEPLOYS) \
        run_overcloud_tests \
        run_undercloud_tests \
        singlebigip \
        setup_singlebigip_tests \
        run_singlebigip_tests \
        cleanup_singlebigip \
        run_disconnected_service_tests \
        run_neutronless_loadbalancer_tests \
        run_neutronless_listener_tests \
        run_neutronless_member_tests \
        run_neutronless_pool_tests

# Don't launch seperate shell proceses on new recipe lines.
.ONESHELL:

# BASH is required for variable indirection:  ${!FOO}
SHELL = /bin/bash

# /ABS/PATH/PROJECT/systest
MAKEFILE_DIR := $(shell dirname $(realpath $(lastword $(MAKEFILE_LIST))))

# /ABS/PATH/PROJECT/systest/scripts
SCRIPT_DIR := $(MAKEFILE_DIR)/scripts

# /ABS/PATH/PROJECT
PROJDIR := $(shell dirname $(realpath $(MAKEFILE_DIR)))

# - <nearest reachable tag>-<num commits since>-g<abbreviated commit id>
TAGINFO := $(shell git describe --long --tags --first-parent)

# git branch

BRANCH := $(shell git rev-parse --abbrev-ref HEAD)

# The branch may contain the text 'stable/' before the OpenStack release name
# To allow this type of branch name and any other to work with the cloud deployment
# and with Jenkins, we should strip out the forward slash, leaving something like:
# stable/newton -> stablenewton
SANITIZED_BRANCH := $(subst /,,$(BRANCH))

# /ASB/PATH/systest/test_results: This is where the CI system will look
# for test reuslts.
RESULTS_DIR := $(MAKEFILE_DIR)/test_results
TESTENVLOGDIR := $(RESULTS_DIR)/f5-openstack-agent_$(SANITIZED_BRANCH)/testenv

#### INITIALIZATION SECTION
#    Since all tests will run on the same CI worker, and we do not (yet)
#    require different packages for different suites, all packages are
#    installed.  Each deploy maps to its own GUMBALLS project.
#    Tests use tox for python variants.
setup:
	-@echo executing $@
	sudo -E -H $(SCRIPT_DIR)/install_test_infra.sh
	mkdir -p $(TESTENVLOGDIR)

#### UNITTEST SECTION:
unit-tests: setup
	-@echo executing $@
	export GP_SUFFIX=f5-openstack-agent_$(SANITIZED_BRANCH)-unit &&
	export GUMBALLS_PROJECT=$(RESULTS_DIR)/$${GP_SUFFIX} &&
	tox -e unit --sitepackages -- \
		--cov $(PROJDIR)/f5_openstack_agent \
		--autolog-outputdir $${GUMBALLS_PROJECT} \
		--autolog-session $(TAGINFO)_`date +"%Y%m%d-%H%M%S"` \
	    $(PROJDIR)/f5_openstack_agent

#### MULTIPLEXING SECTION:
#    there are (DEVICES * CLOUDTYPES) rules in this section. They have targets
#    like '12.1.1-overcloud' and '11.6.1-undercloud'.

#  This rule sets variables to values specific for a particular deploy. This
#  is where interfacing software (trtl) names are set.
#  "GUMBALLS_PROJECT" maps to a raw in gumballs (URL)
#  "GUMBALLS_SESSION" maps to a column in gumballs" (URL)
#  NOTE: GUMBALLS SESSIONs are complete here, e.g.:
#  v8.0.1-435-gab9141e_20170311
$(DEPLOYS): setup
	-@echo executing $@
	. $(MAKEFILE_DIR)/device_version_maps.sh &&
	export BIGIP_VER=`python -c 'print("$@".split("-")[0].replace(".", "_"))'` &&
	export DEVICEVERSION=BIGIP_$${BIGIP_VER} &&
	export TESTENV_NAME=agent_$(SANITIZED_BRANCH)_v$${BIGIP_VER}_bigip &&
	export TESTENV_CONF=bigip.testenv.yaml &&
	export CLOUD=`python -c 'print("$@".split("-")[1])'` &&
	export TIMESTAMP=`date +"%Y%m%d-%H%M%S"` &&
	export GUMBALLS_SESSION=$(TAGINFO)_$${TIMESTAMP} &&
	export GP_SUFFIX=f5-openstack-agent_$(SANITIZED_BRANCH)-$@ &&
	$(MAKE) -C . run_$${CLOUD}_tests

# Currently does nothing in f5-openstack-agent
run_undercloud_tests:
	@echo executing $@
	echo $${CLOUD}

# This rule is run once per device type XX.Y.Z
run_overcloud_tests:
	@echo executing $@
	$(MAKE) -C . setup_singlebigip_tests &&\
	$(MAKE) -C . singlebigip

#### TESTSBYINFRA SECTION
#    Different test scenarios require different infrastructure.  Currently the
#    only infrastructure required by any test is a singlebigip.
#    If a new type of scenario is added that requires a different setup..
#    e.g. a different number of bigips, or realservers, a new RULE will be
#    added to this section.   That scenario will then be appropriately run
#    against all tested DEPLOYS.
#    NOTE: GUMBALLS PROJECTS are complete here, e.g.:
#    singlebigip_f5-openstack-agent_liberty-12.1.1-overcloud
setup_singlebigip_tests:
	@echo executing $@
	echo running testenv create with stack name $${TESTENV_NAME} &&
	testenv --debug create --name $${TESTENV_NAME} \
	        --params bigip_img:$${!DEVICEVERSION} \
	        $${TESTENV_CONF} &> $(TESTENVLOGDIR)/$@_$${TESTENV_NAME}.log

singlebigip:
	@echo executing $@
	export GP_SUFFIX=$@_$${GP_SUFFIX} &&
	export GUMBALLS_PROJECT=$(RESULTS_DIR)/$${GP_SUFFIX} &&
	$(MAKE) -C . run_neutronless_loadbalancer_tests ;\
	$(MAKE) -C . run_neutronless_listener_tests ;\
	$(MAKE) -C . run_neutronless_member_tests ;\
	$(MAKE) -C . run_neutronless_pool_tests ;\
	$(MAKE) -C . run_disconnected_service_tests ;\
	$(MAKE) -C . run_singlebigip_tests ;\
	$(MAKE) -C . cleanup_singlebigip

run_neutronless_loadbalancer_tests:
	@echo executing $@
	tox -e singlebigip --sitepackage -- \
	    --cov $(PROJDIR)/f5_openstack_agent \
		--symbols $(MAKEFILE_DIR)/testenv_symbols/testenv_symbols.json \
		--autolog-outputdir $${GUMBALLS_PROJECT} \
		--autolog-session $${GUMBALLS_SESSION} \
	    ../test/functional/neutronless/loadbalancer/

run_neutronless_listener_tests:
	@echo executing $@
	tox -e singlebigip --sitepackage -- \
	    --cov $(PROJDIR)/f5_openstack_agent \
		--symbols $(MAKEFILE_DIR)/testenv_symbols/testenv_symbols.json \
		--autolog-outputdir $${GUMBALLS_PROJECT} \
		--autolog-session $${GUMBALLS_SESSION} \
	    ../test/functional/neutronless/listener/

run_neutronless_member_tests:
	@echo executing $@
	tox -e singlebigip --sitepackage -- \
	    --cov $(PROJDIR)/f5_openstack_agent \
		--symbols $(MAKEFILE_DIR)/testenv_symbols/testenv_symbols.json \
		--autolog-outputdir $${GUMBALLS_PROJECT} \
		--autolog-session $${GUMBALLS_SESSION} \
	    ../test/functional/neutronless/member/

run_neutronless_pool_tests:
	@echo executing $@
	tox -e singlebigip --sitepackage -- \
	    --cov $(PROJDIR)/f5_openstack_agent \
		--symbols $(MAKEFILE_DIR)/testenv_symbols/testenv_symbols.json \
		--autolog-outputdir $${GUMBALLS_PROJECT} \
		--autolog-session $${GUMBALLS_SESSION} \
	    ../test/functional/neutronless/pool/

run_singlebigip_tests:
	@echo executing $@
	tox -e singlebigip --sitepackage -- \
	    --cov $(PROJDIR)/f5_openstack_agent \
		--symbols $(MAKEFILE_DIR)/testenv_symbols/testenv_symbols.json \
		--autolog-outputdir $${GUMBALLS_PROJECT} \
		--autolog-session $${GUMBALLS_SESSION} \
	    ../test/functional/singlebigip/

# disconnected_singlebigip_f5-openstack-agent_liberty-12_1_1-overcloud
run_disconnected_service_tests:
	@echo executing $@
	export GP_SUFFIX=disconnected_$${GP_SUFFIX} &&
	export GUMBALLS_PROJECT=$(RESULTS_DIR)/$${GP_SUFFIX} &&
	tox -e disconnected_service --sitepackage -- \
	    --cov $(PROJDIR)/f5_openstack_agent \
		--symbols $(MAKEFILE_DIR)/testenv_symbols/testenv_symbols.json \
		--autolog-outputdir $${GUMBALLS_PROJECT} \
		--autolog-session $${GUMBALLS_SESSION} \
	    ../test/functional/neutronless/disconnected_service/

cleanup_singlebigip:
	@echo executing $@
	echo running testenv delete with stack name $${TESTENV_NAME}
	testenv --debug delete --name $${TESTENV_NAME} \
		$${TESTENV_CONF} &> $(TESTENVLOGDIR)/$@_$${TESTENV_NAME}.log
