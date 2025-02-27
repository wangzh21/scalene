LIBNAME = scalene
PYTHON = python3
PYTHON_SOURCES = scalene/[a-z]*.py
C_SOURCES = src/source/libscalene.cpp src/source/get_line_atomic.cpp src/include/*.h*

.PHONY: black clang-format format upload vendor-deps

# CXXFLAGS = -std=c++17 -g -O0 # FIXME
CXXFLAGS = -std=c++17 -g -O3 -DNDEBUG -D_REENTRANT=1 -pipe -fno-builtin-malloc -fvisibility=hidden
CXX = g++

PYTHON_INCLUDE := $(shell python3 -c "from sysconfig import get_paths as gp; print(gp()['include'])")
PYTHON_LIBRARY := $(shell python3 -m find_libpython)

INCLUDES  = -Isrc -Isrc/include
INCLUDES := $(INCLUDES) -Ivendor/Heap-Layers -Ivendor/Heap-Layers/wrappers -Ivendor/Heap-Layers/utility
INCLUDES := $(INCLUDES) -Ivendor/printf
INCLUDES := $(INCLUDES) -I$(PYTHON_INCLUDE)

ifeq ($(shell uname -s),Darwin)
  LIBFILE := lib$(LIBNAME).dylib
  WRAPPER := vendor/Heap-Layers/wrappers/macwrapper.cpp
  ifneq (,$(filter $(shell uname -p),arm arm64))  # this means "if arm or arm64"
    ARCH := -arch arm64 
  else
    ARCH := -arch x86_64
  endif
  CXXFLAGS := $(CXXFLAGS) -flto -ftls-model=initial-exec -ftemplate-depth=1024 $(ARCH) -compatibility_version 1 -current_version 1 -dynamiclib
  RPATH_FLAGS := -Wl,-rpath $(shell dirname $(PYTHON_LIBRARY))
  SED_INPLACE = -i ''

else # non-Darwin
  LIBFILE := lib$(LIBNAME).so
  WRAPPER := vendor/Heap-Layers/wrappers/gnuwrapper.cpp
  INCLUDES := $(INCLUDES) -I/usr/include/nptl 
  CXXFLAGS := $(CXXFLAGS) -fPIC -shared -Bsymbolic
  RPATH_FLAGS :=
  SED_INPLACE = -i

endif

SRC := src/source/lib$(LIBNAME).cpp $(WRAPPER) vendor/printf/printf.cpp

OUTDIR=scalene

all: $(OUTDIR)/$(LIBFILE)

$(OUTDIR)/$(LIBFILE): vendor/Heap-Layers $(SRC) $(C_SOURCES) GNUmakefile
	$(CXX) $(CXXFLAGS) $(INCLUDES) $(SRC) -o $(OUTDIR)/$(LIBFILE) -ldl -lpthread $(RPATH_FLAGS) $(PYTHON_LIBRARY)

clean:
	rm -f $(OUTDIR)/$(LIBFILE) scalene/get_line_atomic*.so
	rm -rf $(OUTDIR)/$(LIBFILE).dSYM
	rm -rf scalene.egg-info
	rm -rf build dist *egg-info

$(WRAPPER) : vendor/Heap-Layers

vendor/Heap-Layers:
	mkdir -p vendor && cd vendor && git clone https://github.com/emeryberger/Heap-Layers

vendor/printf/printf.cpp:
	mkdir -p vendor && cd vendor && git clone https://github.com/mpaland/printf
	cd vendor/printf && ln -s printf.c printf.cpp
	sed $(SED_INPLACE) -e 's/^#define printf printf_/\/\/&/' vendor/printf/printf.h

vendor-deps: vendor/Heap-Layers vendor/printf/printf.cpp

mypy:
	-mypy $(PYTHON_SOURCES)

format: black clang-format

clang-format:
	-clang-format -i $(C_SOURCES) --style=google

black:
	-black -l 79 $(PYTHON_SOURCES)

#ifeq ($(shell uname -s),Darwin)
#  PYTHON_PLAT:=-p $(shell $(PYTHON) -c 'from pkg_resources import get_build_platform; p=get_build_platform(); print(p[:p.rindex("-")])')-universal2
#endif

PYTHON_API_VER:=$(shell $(PYTHON) -c 'from pip._vendor.packaging.tags import interpreter_name, interpreter_version; print(interpreter_name()+interpreter_version())')

bdist: vendor-deps
	$(PYTHON) setup.py bdist_wheel $(PYTHON_PLAT)
ifeq ($(shell uname -s),Linux)
	auditwheel repair dist/*.whl
	rm -f dist/*.whl
	mv wheelhouse/*.whl dist/
endif

sdist: vendor-deps
	$(PYTHON) setup.py sdist

upload: sdist bdist # to pypi
	$(PYTHON) -m twine upload dist/*
