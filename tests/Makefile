
# To compile on SunOS: add "-lsocket -lnsl" to LDFLAGS
# To compile with PKCS11: add "-lpkcs11-helper" to LDFLAGS

CFLAGS	?= -O2
WARNING_CFLAGS ?= -Wall -W -Wdeclaration-after-statement -Wunused
LDFLAGS ?=

# Check test environment. If ../library is available then Mbed TLS is used.
# Otherwise Mbed OS environment is used.
DIR_FOR_MBED_TLS_ENV=../library
ifneq "$(wildcard $(DIR_FOR_MBED_TLS_ENV) )" ""
	LIBRARY_DIR=../library
	INCLUDE_DIR=-I../include
else
	LIBRARY_DIR=../src
	INCLUDE_DIR=-I../inc
	CFLAGS += "-DMBEDTLS_CONFIG_FILE=\"mbedtls/test_config.h\""
	CFLAGS += "-DTEST_ENV_MBED_OS"
endif

LOCAL_CFLAGS = $(WARNING_CFLAGS) $(INCLUDE_DIR) -D_FILE_OFFSET_BITS=64

LOCAL_LDFLAGS = -L$(LIBRARY_DIR)		\
		-lmbedtls$(SHARED_SUFFIX)	\
		-lmbedx509$(SHARED_SUFFIX)	\
		-lmbedcrypto$(SHARED_SUFFIX)

# Enable definition of various functions used throughout the testsuite
# (gethostname, strdup, fileno...) even when compiling with -std=c99. Harmless
# on non-POSIX platforms.
LOCAL_CFLAGS += -D_POSIX_C_SOURCE=200809L

ifndef SHARED
DEP=$(LIBRARY_DIR)/libmbedcrypto.a $(LIBRARY_DIR)/libmbedx509.a $(LIBRARY_DIR)/libmbedtls.a
else
DEP=$(LIBRARY_DIR)/libmbedcrypto.$(DLEXT) $(LIBRARY_DIR)/libmbedx509.$(DLEXT) $(LIBRARY_DIR)/libmbedtls.$(DLEXT)
endif

ifdef DEBUG
LOCAL_CFLAGS += -g3
endif

# if we're running on Windows, build for Windows
ifdef WINDOWS
WINDOWS_BUILD=1
endif

ifdef WINDOWS_BUILD
DLEXT=dll
EXEXT=.exe
LOCAL_LDFLAGS += -lws2_32
ifdef SHARED
SHARED_SUFFIX=.$(DLEXT)
endif
PYTHON ?= python
else
DLEXT ?= so
EXEXT=
SHARED_SUFFIX=
# python2 for POSIX since FreeBSD has only python2 as default.
PYTHON ?= python2
endif

# Zlib shared library extensions:
ifdef ZLIB
LOCAL_LDFLAGS += -lz
endif

# Pthread shared library extension
ifdef PTHREAD
LOCAL_LDFLAGS += -lpthread
endif

# A test application is built for each suites/test_suite_*.data file.
# Application name is same as .data file's base name and can be
# constructed by stripping path 'suites/' and extension .data.
APPS = $(basename $(subst suites/,,$(wildcard suites/test_suite_*.data)))

ifndef PTHREAD
APPS := $(filter-out test_suite_x509parse_pthread, $(APPS))
endif

# Construct executable name by adding OS specific suffix $(EXEXT).
BINARIES := $(addsuffix $(EXEXT),$(APPS))

.SILENT:

.PHONY: all check test clean

all: $(BINARIES)

$(DEP):
	$(MAKE) -C $(LIBRARY_DIR) $(INCLUDE_DIR)

C_FILES := $(addsuffix .c,$(APPS))

# Wildcard target for test code generation:
# A .c file is generated for each .data file in the suites/ directory. Each .c
# file depends on a .data and .function file from suites/ directory. Following
# nameing convention is followed:
#
#     C file        |        Depends on
#-----------------------------------------------------------------------------
#  foo.c            | suites/foo.function suites/foo.data
#  foo.bar.c        | suites/foo.function suites/foo.bar.data
#
# Note above that .c and .data files have same base name.
# However, corresponding .function file's base name is the word before first
# dot in .c file's base name.
#
.SECONDEXPANSION:
%.c: suites/$$(firstword $$(subst ., ,$$*)).function suites/%.data scripts/generate_test_code.py suites/helpers.function suites/main_test.function suites/host_test.function
	echo "  Gen   $@"
	$(PYTHON) scripts/generate_test_code.py -f suites/$(firstword $(subst ., ,$*)).function \
		-d suites/$*.data \
		-t suites/main_test.function \
		-p suites/host_test.function \
		-s suites  \
		--helpers-file suites/helpers.function \
		-o .


$(BINARIES): %$(EXEXT): %.c $(DEP)
	echo "  CC    $<"
	$(CC) $(LOCAL_CFLAGS) $(CFLAGS) $<	$(LOCAL_LDFLAGS) $(LDFLAGS) -o $@


clean:
ifndef WINDOWS
	rm -rf $(BINARIES) *.c *.su *.datax TESTS
else
	del /Q /F *.c *.su *.exe *.datax
ifneq ($(wildcard TESTS/.*),)
	rmdir /Q /S TESTS
endif
endif

# Test suites caught by SKIP_TEST_SUITES are built but not executed.
check: $(BINARIES)
	perl scripts/run-test-suites.pl --skip=$(SKIP_TEST_SUITES)

test: check

# Create separate targets for generating embedded tests.
EMBEDDED_TESTS := $(addprefix embedded_,$(APPS))

# Generate test code for target.

.SECONDEXPANSION:
$(EMBEDDED_TESTS): embedded_%: suites/$$(firstword $$(subst ., ,$$*)).function suites/%.data scripts/generate_test_code.py suites/helpers.function suites/main_test.function suites/target_test.function
	echo "  Gen  ./TESTS/mbedtls/$*/$*.c"
	$(PYTHON) scripts/generate_test_code.py -f suites/$(firstword $(subst ., ,$*)).function \
		-d suites/$*.data \
		-t suites/main_test.function \
		-p suites/target_test.function \
		-s suites  \
		--helpers-file suites/helpers.function \
		-o ./TESTS/mbedtls/$*

generate-target-tests: $(EMBEDDED_TESTS)
