MAKDIR := $(subst //,,$(dir $(firstword $(MAKEFILE_LIST)))/)
INCACC := $(wildcard $(MAKDIR)/*.h*) $(MAKDIR)/../acc.h
SRCACC := $(wildcard $(MAKDIR)/*.c)
OBJACC := $(SRCACC:.c=.o)

INCSMM := $(wildcard $(MAKDIR)/smm/*.h*) \
                     $(MAKDIR)/smm/opencl_kernels.h \
                     $(MAKDIR)/../acc_libsmm.h \
                     $(MAKDIR)/../acc_bench.h \
                     $(NULL)
SRCSMM := $(wildcard $(MAKDIR)/smm/*.c)
OBJSMM := $(SRCSMM:.c=.o)
KERNEL := $(wildcard $(MAKDIR)/smm/kernels/*.cl)

INCALL := $(INCACC) $(INCSMM)

LIBXSMMROOT := $(wildcard $(MAKDIR)/../../../../libxsmm)
ifeq (,$(LIBXSMMROOT))
  LIBXSMMROOT := $(wildcard $(HOME)/libxsmm)
endif
UNAME := $(shell uname)
INTEL ?= 0
DEV ?= 0

# select from set of predefined triplet specifications
SPECID ?= 0
# limit shape in tests (zero or negative for unlimited)
MAXEXT ?= 48
# number of tests (zero or negative for unlimited)
NTRANS ?= 10
NSMMS ?= 10

COMMAND := $(shell which command 2>/dev/null)
ifneq (,$(COMMAND))
  which = $(shell $(COMMAND) -v $1)
else
  which = $(shell which $(firstword $1) 2>/dev/null)
endif

WITH_GPU := $(if $(WITH_GPU),$(WITH_GPU),$(GPUVER))
PARAMS_WITHGPU := $(MAKDIR)/smm/params/tune_multiply_$(WITH_GPU).csv
PARAMS_DEFAULT := $(MAKDIR)/smm/tune_multiply.csv
PARAMS := $(if $(wildcard $(PARAMS_WITHGPU)),$(PARAMS_WITHGPU),$(wildcard $(PARAMS_DEFAULT)))

#PARAMS_DIR ?= $(MAKDIR)/smm/params
ifeq (,$(PARAMS))
ifneq (,$(wildcard $(PARAMS_DIR)))
  WITH_GPUS := $(shell ls -1 $(PARAMS_DIR)/*.csv | cut -d. -f1 | rev | cut -d_ -f1 | rev)
endif
endif

CFLAGS := -fPIC \
  -Wall -Wextra -pedantic \
  -Wno-overlength-strings \
  -Wno-variadic-macros \
  -Wno-unused-function \
  -Wno-long-long \
  -D__OPENCL \
  $(NULL)

ifneq (,$(ELEM_TYPE))
  CFLAGS += -DELEM_TYPE=$(ELEM_TYPE)
endif

ifeq (1,$(INTEL))
  CXX := icpc
  CC := icc
  AR := xiar
else ifneq (0,$(INTEL))
  CXX := icpx
  CC := icx
  AR := xiar
else
  CXX := g++
  CC := gcc
  ifneq (Darwin,$(UNAME))
    AR := gcc-ar
  else
    AR := ar
  endif
endif

ifneq (0,$(DEV))
  ifeq (1,$(DEV))
    CFLAGS := -std=c89 $(CFLAGS)
    CFLAGS += -Wno-unused-parameter
  else
    CFLAGS := -Wno-deprecated -Werror $(CFLAGS)
    ifneq (,$(findstring clang,$(CC) $(CXX)))
      override CC := clang++ --analyze
    else
      CC := $(CXX)
    endif
    OMP := 0
  endif
else
  CFLAGS := -std=c99 $(CFLAGS)
endif

ifneq (0,$(DBG))
  CPP_OPENCL_FLAGS += -C
  ifeq (,$(DBG))
    CFLAGS += -O2 -DNDEBUG
  else
    ifneq (1,$(DBG))
      CFLAGS += -D_DEBUG
    endif
    CFLAGS += -O0
  endif
else
  CFLAGS += -O2 -DNDEBUG -DNDBGDEV
  SYM := 0
endif
ifneq (0,$(SYM))
  CFLAGS += -g
endif

ifneq (0,$(OMP))
  ifneq (0,$(INTEL))
    CFLAGS += -qopenmp
    LDFLAGS += -qopenmp
  else ifneq (Darwin,$(UNAME))
    CFLAGS += -fopenmp
    LDFLAGS += -fopenmp
  else # macOS
    CFLAGS += -Xpreprocessor -fopenmp
    LDFLAGS += -lomp
  endif
  ifneq (,$(LIBXSMMROOT))
    LDFLAGS += -lxsmmext -lxsmm -lxsmmnoblas -ldl -lm
  endif
else
  ifneq (,$(LIBXSMMROOT))
    LDFLAGS += -lxsmm -lxsmmnoblas -ldl -lm
  endif
endif
ifneq (,$(LIBXSMMROOT))
  CFLAGS_XSMM += -pthread -D__LIBXSMM -I$(LIBXSMMROOT)/include
  LDFLAGS := -pthread $(LDFLAGS)
  LDFLAGS += -L$(LIBXSMMROOT)/lib
  ifneq (Darwin,$(UNAME))
    LDFLAGS += -Wl,-rpath=$(LIBXSMMROOT)/lib $(LDFLAGS)
  endif
endif

ifeq (Darwin,$(UNAME))
  LDFLAGS += -framework OpenCL
else
  ifeq (,$(CUDATOOLKIT_HOME))
    CUDATOOLKIT_HOME := $(NVSDKCOMPUTE_ROOT)
  endif
  ifeq (,$(CUDATOOLKIT_HOME))
    NVCC := $(call which,nvcc)
    CUDATOOLKIT_HOME := $(if $(NVCC),$(abspath $(dir $(NVCC))/..))
  endif
  ifneq (,$(CUDATOOLKIT_HOME))
    CFLAGS += -I$(CUDATOOLKIT_HOME)/include
    ifeq (,$(wildcard $(LIBOPENCL)))
      LDFLAGS += -L$(CUDATOOLKIT_HOME)/lib64
      LDFLAGS += -Wl,-rpath=$(CUDATOOLKIT_HOME)/lib64
    endif
  endif
  ifeq (,$(wildcard $(LIBOPENCL)))
    LDFLAGS += -lOpenCL
  else
    LDFLAGS += $(LIBOPENCL)
  endif
endif

CXXLIBDIR := $(dir $(call which,$(CXX)))/../lib64
ifneq (,$(wildcard $(CXXLIBDIR)))
  LDFLAGS += -Wl,-rpath=$(abspath $(CXXLIBDIR))
endif

.PHONY: bench
bench: $(MAKDIR)/../acc_bench_smm $(MAKDIR)/../acc_bench_trans

.PHONY: all
all: bench $(MAKDIR)/../dbcsr_acc_test

.PHONY: test
test: test-interface test-trans test-smm

.PHONY: test-interface
test-interface: $(MAKDIR)/../dbcsr_acc_test
	@echo "--- DBCSR Backend Interface"
	$(MAKDIR)/../dbcsr_acc_test

.PHONY: test-trans
test-trans: bench
	$(eval SHAPES = $(shell $(MAKDIR)/../acc_triplets.sh -s $(SPECID) -m $(MAXEXT) -n $(NTRANS) -a))
	$(eval DEVICE = $(shell ACC_OPENCL_VERBOSE=1 CHECK=0 $(MAKDIR)/../acc_bench_smm 1 1 1 2>&1 >/dev/null))
	@echo "--- DBCSR OpenCL Transposes ($(words $(SHAPES)))"
	@echo "$(DEVICE)"
	@echo "LD_LIBRARY_PATH=${LD_LIBRARY_PATH}"
ifneq (,$(LD_PRELOAD))
	@echo "LD_PRELOAD=${LD_PRELOAD}"
endif
	@echo "CC: $$($(CC) --version | head -n1)"
	@echo "runtime libraries:"
	@ldd $(MAKDIR)/../acc_bench_trans
	@echo "hostname: $$(hostname)"
	@echo
	@for SHAPE in $(SHAPES); do \
		$(MAKDIR)/../acc_bench_trans $${SHAPE} || exit 1; \
		echo; \
	done

$(MAKDIR)/test-smm.log: bench
	$(eval SHAPES = $(shell $(MAKDIR)/../acc_triplets.sh -s $(SPECID) -m $(MAXEXT) -n $(NSMMS)))
	$(eval DEVICE = "$(shell LIBXSMM_VERBOSE=0 ACC_OPENCL_VERBOSE=1 CHECK=0 $(MAKDIR)/../acc_bench_smm 1 1 1 2>&1 >/dev/null)")
	$(eval WITH_GPU = $(firstword $(foreach GPU,$(WITH_GPUS),$(findstring $(GPU),$(DEVICE)))))
	$(eval PARAMS = $(firstword $(wildcard $(PARAMS_DIR)/tune_multiply_*$(WITH_GPU).csv)))
	$(eval GPUENV = $(if $(PARAMS),$(if $(OPENCL_LIBSMM_SMM_PARAMS),$(NULL),OPENCL_LIBSMM_SMM_PARAMS=$(PARAMS))))
	@echo "--- DBCSR OpenCL SMMs ($(words $(SHAPES)))"
	@echo "$(DEVICE)"
	@if [ "$(GPUENV)" ]; then echo "$(GPUENV)"; fi
	@echo "LD_LIBRARY_PATH=${LD_LIBRARY_PATH}"
ifneq (,$(LD_PRELOAD))
	@echo "LD_PRELOAD=${LD_PRELOAD}"
endif
	@echo "CC: $$($(CC) --version | head -n1)"
	@echo "runtime libraries:"
	@ldd $(MAKDIR)/../acc_bench_smm
	@echo "hostname: $$(hostname)"
	@echo
	@echo "$(SHAPES)" | xargs -n1 | \
		($(GPUENV) CHECK=$(if $(CHECK),$(CHECK),1) stdbuf --output=L $(MAKDIR)/../acc_bench_smm /dev/stdin \
			2>$(MAKDIR)/test-smm.err && rm $(MAKDIR)/test-smm.err) | tee $@
	@if [ -s $(MAKDIR)/test-smm.err ]; then cat $(MAKDIR)/test-smm.err && exit 1; fi

.PHONY: test-smm
test-smm: $(MAKDIR)/test-smm.log
ifneq (,$(call which,datamash))
ifeq (,$(shell datamash geomean 2>&1 | grep invalid))
	@echo "geomean: $$(sed -n "/device:/p" $< | datamash -W -R 1 geomean 4) GFLOPS/s"
endif
	@echo "median: $$(sed -n "/device:/p" $< | datamash -W -R 1 median 4) GFLOPS/s"
	@echo "mean: $$(sed -n "/device:/p" $< | datamash -W -R 1 mean 4) GFLOPS/s"
endif

$(MAKDIR)/smm/opencl_kernels.h: $(MAKDIR)/acc_opencl.sh $(KERNEL) $(PARAMS)
	@CPPFLAGS=$(CPP_OPENCL_FLAGS) $(MAKDIR)/acc_opencl.sh $(KERNEL) $(PARAMS) $@

$(MAKDIR)/../dbcsr_acc.a: $(OBJACC)
	$(AR) -rs $@ $^

$(MAKDIR)/../dbcsr_acc_smm.a: $(OBJSMM)
	$(AR) -rs $@ $^

%.o: %.c $(INCALL) $(MAKDIR)/Makefile
	$(CC) $(CFLAGS) $(CFLAGS_XSMM) -c $< -o $@

$(MAKDIR)/acc_bench_smm.o: $(MAKDIR)/../acc_bench_smm.c $(MAKDIR)/Makefile
ifneq (0,$(LIBXSMM))
	$(CC) $(CFLAGS) $(CFLAGS_XSMM) -c $< -o $@
else
	$(CC) $(CFLAGS) -c $< -o $@
endif

$(MAKDIR)/../acc_bench_smm: $(MAKDIR)/acc_bench_smm.o $(MAKDIR)/../dbcsr_acc_smm.a $(MAKDIR)/../dbcsr_acc.a
ifneq (,$(filter 0 1,$(DEV)))
	$(CC) $^ $(LDFLAGS) -o $@
else
.PHONY: $(MAKDIR)/../acc_bench_smm
endif

$(MAKDIR)/acc_bench_trans.o: $(MAKDIR)/../acc_bench_trans.c $(MAKDIR)/Makefile
ifneq (0,$(LIBXSMM))
	$(CC) $(CFLAGS) $(CFLAGS_XSMM) -c $< -o $@
else
	$(CC) $(CFLAGS) -c $< -o $@
endif

$(MAKDIR)/../acc_bench_trans: $(MAKDIR)/acc_bench_trans.o $(MAKDIR)/../dbcsr_acc_smm.a $(MAKDIR)/../dbcsr_acc.a
ifneq (,$(filter 0 1,$(DEV)))
	$(CC) $^ $(LDFLAGS) -o $@
else
.PHONY: $(MAKDIR)/../acc_bench_trans
endif

$(MAKDIR)/dbcsr_acc_test.o: $(MAKDIR)/../../../tests/dbcsr_acc_test.c $(MAKDIR)/Makefile
	$(CC) $(CFLAGS) -I$(MAKDIR)/../.. -c $< -o $@

$(MAKDIR)/../dbcsr_acc_test: $(MAKDIR)/dbcsr_acc_test.o $(MAKDIR)/../dbcsr_acc.a
ifneq (,$(filter 0 1,$(DEV)))
	$(CC) $^ $(LDFLAGS) -o $@
else
.PHONY: $(MAKDIR)/../dbcsr_acc_test
endif

.PHONY: clean
clean:
	@rm -f $(OBJACC) $(OBJSMM)
	@rm -f $(MAKDIR)/dbcsr_acc_test.o
	@rm -f $(MAKDIR)/acc_bench_trans.o
	@rm -f $(MAKDIR)/acc_bench_smm.o
	@rm -f $(MAKDIR)/smm/opencl_kernels.h
	@rm -f $(MAKDIR)/test-smm.err

.PHONY: realclean
realclean: clean
	@rm -f $(MAKDIR)/../dbcsr_acc.a $(MAKDIR)/../dbcsr_acc_smm.a
	@rm -f $(MAKDIR)/../acc_bench_smm $(MAKDIR)/../acc_bench_trans
	@rm -f $(MAKDIR)/../dbcsr_acc_test
	@rm -f $(MAKDIR)/test-smm.log
