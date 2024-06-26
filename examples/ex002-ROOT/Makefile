BUILD_DIR=build
UNAME=$(shell uname)

JL_SHARE = $(shell julia -e 'print(joinpath(Sys.BINDIR, Base.DATAROOTDIR, "julia"))')
CXXFLAGS += $(patsubst -std=gnu%,,$(shell $(JL_SHARE)/julia-config.jl --cflags))
#CXXFLAGS += -DVERBOSE_IMPORT #To enable the verbose mode of the libray loading
#CXXFLAGS += -Wall -O0 -g     #To compile with debugger infomation
LDFLAGS  += $(shell $(JL_SHARE)/julia-config.jl --ldflags)
LDLIBS = $(shell $(JL_SHARE)/julia-config.jl --ldlibs) -L$(CXXWRAP_PREFIX)/lib -lcxxwrap_julia -lcxxwrap_julia_stl
#LDLIBS   += $(shell $(JL_SHARE)/julia-config.jl --ldlibs)
WRAPIT = $(shell which wrapit)
WIT_FILE = ROOT.wit
CXXWRAP_VERSION=$(shell julia -e "import TOML; print(get(TOML.parse(open(\"$(WIT_FILE)\")), \"cxxwrap_version\", \"\"));")

ifeq ($(CXXWRAP_VERSION),)
CXXWRAP_PREFIX=$(shell mkdir -p $(BUILD_DIR); julia --project=$(BUILD_DIR) -e "import Pkg; Pkg.add(\"CxxWrap\"); Pkg.resolve(); import CxxWrap; print(CxxWrap.prefix_path())")
CXXWRAP_VERSION:=$(shell julia --project=${BUILD_DIR} -e "import CxxWrap; print(pkgversion(CxxWrap));")
WRAPIT_OPT=--add-cfg cxxwrap_version=\"$(CXXWRAP_VERSION)\"
else
CXXWRAP_PREFIX=$(shell mkdir -p $(BUILD_DIR); julia --project=$(BUILD_DIR) -e "import Pkg; Pkg.add(name=\"CxxWrap\", version=\"$(CXXWRAP_VERSION)\"); Pkg.resolve(); import CxxWrap; print(CxxWrap.prefix_path())")
endif

CXXWRAP_CPPFLAGS=-I $(CXXWRAP_PREFIX)/include -I . --std=c++20
LDLIBS   += $(shell $(JL_SHARE)/julia-config.jl --ldlibs)

CXX_IS_CLANG=$(shell $(CXX) -dM -E - < /dev/null | grep -c __clang__)

ifeq ($(UNAME),Darwin)
  SO_SUFFIX = .dylib
else
  SO_SUFFIX = .so
endif

ifneq ($(CXX_IS_CLANG), 0)
	CXXFLAGS += -ferror-limit=3
else #assuming gcc
	CXXFLAGS += -fmax-errors=3
endif

CXXFLAGS += -O2

CPPFLAGS += -MMD -I $(ROOT_INC_DIR) -I.

ROOT_LIBS = $(shell root-config --libs)
ROOT_INC_DIR = $(shell root-config --incdir)

LINK.o = $(CXX) $(LDFLAGS) $(TARGET_ARCH)

.PHONY: all clean run_demo run_demo2 check_root make_lib_from_objs

.PRECIOUS: $(GENERATED_CXX) $(BUILD_DIR)/ROOT-generated.wit

PRODUCTS=$(addprefix $(BUILD_DIR)/, ROOT/deps/libjlROOT$(SO_SUFFIX) ROOT-generated.wit ROOT/src/ROOT.jl ROOT/src/iROOT.jl ROOT/src/ROOTex.jl ROOT/Project.toml ROOT/src/ROOT-export.jl)

LOGS=$(BUILD_DIR)/jlROOT-report.txt
#GENERATED_CXX:=$(file < $(BUILD_DIR)/libROOT/src/generated_cxx)
GENERATED_CXX:=$(shell cat $(BUILD_DIR)/libROOT/src/generated_cxx)
OBJS=$(addprefix $(BUILD_DIR)/libROOT/build/, $(patsubst %.cxx,%.o, $(GENERATED_CXX)))
DEPS=$(patsubst %.o,%.d, $(OBJS))

all: $(PRODUCTS)

clean:
	-$(RM) -r build

$(BUILD_DIR):
	@mkdir $(BUILD_DIR)
	@echo 'all:\n\n%:\n\t$$(MAKE) -C .. $$@\nclean:\n\t$$(MAKE) -C .. clean build\n' > $(BUILD_DIR)/Makefile

$(BUILD_DIR)/ROOT-generated.wit: ROOT.wit $(BUILD_DIR)
	$(MAKE) check_root
	$(shell echo "#\n# Warning: file generated automatically from $<\n#" > $@)
	$(shell sed "s@%ROOT_INC_DIR%@$(ROOT_INC_DIR)@" $< >> $@ || rm $@)

check_root:
ifeq ($(ROOT_LIBS),)
	$(error ERROR: "Command root-config not found. ROOT (http:://root.cern.ch) environment needs to be set")
endif

run_demo: all
	LD_LIBRARY_PATH=libROOT:$(shell root-config --libdir) JULIA_LOAD_PATH=.:@:@v#.#:@stdlib julia -i demo_ROOT.jl

run_demo2: all
	LD_LIBRARY_PATH=libROOT:$(shell root-config --libdir) JULIA_LOAD_PATH=.:@:@v#.#:@stdlib julia -i demo_TGraph.jl

test: all
	LD_LIBRARY_PATH=libROOT:$(shell root-config --libdir) \
JULIA_LOAD_PATH=`pwd`/build/ROOT/src:$(JULIA_LOAD_PATH): julia --project=$(BUILD_DIR) demo_ROOT.jl
	cmp demo_ROOT.png demo_ROOT-ref.png
	LD_LIBRARY_PATH=libROOT:$(shell root-config --libdir) \
JULIA_LOAD_PATH=`pwd`/build/ROOT/src:$(JULIA_LOAD_PATH): julia --project=$(BUILD_DIR) demo_TGraph.jl
	cmp demo_TGraph.png demo_TGraph-ref.png
	LD_LIBRARY_PATH=libROOT:$(shell root-config --libdir) \
JULIA_LOAD_PATH=`pwd`/build/ROOT/src:$(JULIA_LOAD_PATH): julia --project=$(BUILD_DIR) TTree_examples/write_tree1.jl \
&& JULIA_LOAD_PATH=`pwd`/build/ROOT/src:$(JULIA_LOAD_PATH): julia --project=$(BUILD_DIR) TTree_examples/read_tree1.jl
	LD_LIBRARY_PATH=libROOT:$(shell root-config --libdir) \
JULIA_LOAD_PATH=`pwd`/build/ROOT/src:$(JULIA_LOAD_PATH): julia --project=$(BUILD_DIR) TTree_examples/write_tree2.jl \
&& JULIA_LOAD_PATH=`pwd`/build/ROOT/src:$(JULIA_LOAD_PATH): julia --project=$(BUILD_DIR) -e 'import Pkg; Pkg.activate(;temp=true); Pkg.add("UnROOT"); include("TTree_examples/read_tree2.jl")'
	LD_LIBRARY_PATH=libROOT:$(shell root-config --libdir) \
JULIA_LOAD_PATH=`pwd`/build/ROOT/src:$(JULIA_LOAD_PATH): julia --project=$(BUILD_DIR) TTree_examples/write_tree3.jl \
&& JULIA_LOAD_PATH=`pwd`/build/ROOT/src:$(JULIA_LOAD_PATH): julia --project=$(BUILD_DIR) TTree_examples/read_tree3.jl

%/libROOT/src/generated_cxx %/ROOT/Project.toml %/ROOT/src/ROOT-export.jl: %/ROOT-generated.wit jlROOT-veto.h $(WRAPIT) TBranchPtr.h Templates.h
	$(MAKE) $*/ROOT-generated.wit
	$(WRAPIT) $(WRAPIT_OPT) --force --update --output-prefix $* $<
	$(eval GENERATED_CXX:=$(file < $(BUILD_DIR)/libROOT/src/generated_cxx))
	$(eval OBJS:=$(addprefix $(BUILD_DIR)/libROOT/build/, $(patsubst %.cxx,%.o, $(GENERATED_CXX))))

$(BUILD_DIR)/libROOT/build/%.o: $(BUILD_DIR)/libROOT/src/%.cxx $(BUILD_DIR)
	[ -d $(BUILD_DIR)/libROOT/build ] || mkdir -p $(BUILD_DIR)/libROOT/build
	$(COMPILE.cc) $(CXXWRAP_CPPFLAGS) -o $@ $<

$(BUILD_DIR)/ROOT/deps/libjlROOT$(SO_SUFFIX): $(BUILD_DIR)/libROOT/src/generated_cxx $(OBJS)
	$(MAKE) check_root
	$(MAKE) $<
	$(MAKE) make_lib_from_objs

$(BUILD_DIR)/ROOT/src/%.jl: %.jl  $(BUILD_DIR)
	[ -d $(BUILD_DIR)/ROOT/src ] || mkdir -p $(BUILD_DIR)/ROOT/src
	cp -a $< $(BUILD_DIR)/ROOT/src/


make_lib_from_objs: $(OBJS) $(BUILD_DIR)
	[ -d $(BUILD_DIR)/ROOT/deps ] || mkdir -p $(BUILD_DIR)/ROOT/deps
	$(LINK.o) -o $(BUILD_DIR)/ROOT/deps/libjlROOT$(SO_SUFFIX) --shared -fPIC $(OBJS) $(ROOT_LIBS) $(LDLIBS)

echo_%:
	@echo "$* = $(subst ",\",$($*))"

-include $(DEPS)
