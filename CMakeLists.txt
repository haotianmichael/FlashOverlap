cmake_minimum_required(VERSION 3.18 FATAL_ERROR)
project(flashoverlap LANGUAGES CXX CUDA)

if (DEFINED ENV{CONDA_PREFIX})
  # use conda environment
  link_directories($ENV{CONDA_PREFIX}/lib)
  include_directories($ENV{CONDA_PREFIX}/include)
endif()

find_package(CUDAToolkit 11.4 REQUIRED)

# gcc >= 8 is required, we do not support other compilers
if ((NOT CMAKE_CXX_COMPILER_ID STREQUAL "GNU") OR (CMAKE_CXX_COMPILER_VERSION VERSION_LESS 8.0))
    message(FATAL_ERROR "GCC 8.0 or higher is required")
endif()
# Add filesystem library for gcc < 9
link_libraries( "$<$<AND:$<CXX_COMPILER_ID:GNU>,$<VERSION_LESS:$<CXX_COMPILER_VERSION>,9.0>>:-lstdc++fs>" )

# Set up C++ standard
# [modified] Recent PyTorch headers require C++20 (was 17).
#            Also apply the standard to CUDA sources, otherwise nvcc stays at its default (C++17).
set(CXX_STD "20" CACHE STRING "C++ standard")
set(CMAKE_CXX_STANDARD ${CXX_STD})
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CUDA_STANDARD ${CXX_STD})
set(CMAKE_CUDA_STANDARD_REQUIRED ON)

if (BUILD_MODE)
  string(TOUPPER ${BUILD_MODE} BUILD_MODE)
  if (BUILD_MODE STREQUAL "DEBUG")
    set(DEBUG ON)
  elseif (BUILD_MODE STREQUAL "RELEASE")
    set(RELEASE ON)
  else()
    message(FATAL_ERROR "Unknown build mode: ${BUILD_MODE}")
  endif()
else()
  message("No build type selected, defaulting to RELEASE mode")
  message("Use -DBUILD_MODE=DEBUG or -DBUILD_MODE=RELEASE to specify build type")
  set(RELEASE ON)
endif()

set(COMMON_HEADER_DIRS
  ${PROJECT_SOURCE_DIR}/src
)
set(COMMON_LIB_DIRS "")

# [modified] Moved up from the Python section: needed earlier to locate pip-installed NCCL.
set(PYTHON_PATH "python" CACHE STRING "Python path")

# Set up MPI and NCCL for multi-GPU communication
message("Building with MPI and NCCL")
set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/Modules)

# [added] Prefer the NCCL shipped with PyTorch (pip package nvidia-nccl-*), so this
#         project and torch load the same libnccl.so.2. Skipped if the user passes
#         -DNCCL_INCLUDE_DIRS / -DNCCL_LIBRARIES; falls back to FindNCCL's normal search
#         if the package is not installed.
if (NOT NCCL_INCLUDE_DIRS OR NOT NCCL_LIBRARIES)
  execute_process(COMMAND ${PYTHON_PATH} "-c" "import nvidia.nccl as m; print(list(m.__path__)[0], end='')"
                  RESULT_VARIABLE _NCCL_PY_RESULT
                  OUTPUT_VARIABLE _NCCL_PY_DIR
                  ERROR_QUIET)
  if (_NCCL_PY_RESULT EQUAL 0
      AND EXISTS "${_NCCL_PY_DIR}/include/nccl.h"
      AND EXISTS "${_NCCL_PY_DIR}/lib/libnccl.so.2")
    message(STATUS "Using NCCL from Python package: ${_NCCL_PY_DIR}")
    if (NOT NCCL_INCLUDE_DIRS)
      set(NCCL_INCLUDE_DIRS "${_NCCL_PY_DIR}/include" CACHE PATH "NCCL include directory" FORCE)
    endif()
    if (NOT NCCL_LIBRARIES)
      set(NCCL_LIBRARIES "${_NCCL_PY_DIR}/lib/libnccl.so.2" CACHE FILEPATH "NCCL library" FORCE)
    endif()
  endif()
endif()

# set(MKL_MPI "openmpi")
find_package(NCCL REQUIRED)
# find_package(MPI REQUIRED)
set(CMAKE_MODULE_PATH "") # prevent the bugs for pytorch building

# [added] The pip package only ships libnccl.so.2 (no libnccl.so), so '-lnccl' cannot
#         resolve. Provide a libnccl.so symlink inside the build tree (site-packages untouched).
set(NCCL_LINK_DIR ${CMAKE_BINARY_DIR}/nccl_link)
file(MAKE_DIRECTORY ${NCCL_LINK_DIR})
file(CREATE_LINK ${NCCL_LIBRARIES} ${NCCL_LINK_DIR}/libnccl.so SYMBOLIC)

# [modified] NCCL_INCLUDE_DIR -> NCCL_INCLUDE_DIRS: the former is only an (usually empty)
#            input hint of FindNCCL; the latter is its result.
list(APPEND COMMON_HEADER_DIRS ${MPI_INCLUDE_PATH} ${NCCL_INCLUDE_DIRS})
# [modified] link_directories() expects directories, not library files.
list(APPEND COMMON_LIB_DIRS ${MPI_LIBRARIES} ${NCCL_LINK_DIR})

set(COMMON_LIBS CUDA::cudart)
# [added] Link NCCL by full path; CMake then also puts its directory into the build RPATH.
list(APPEND COMMON_LIBS ${NCCL_LIBRARIES})

# Add Python into COMMON_HEADER_DIRS & COMMON_LIB_DIRS
execute_process(COMMAND ${PYTHON_PATH} "-c" "import sysconfig;
print(sysconfig.get_paths()['include']);"
                  RESULT_VARIABLE _PYTHON_SUCCESS
                  OUTPUT_VARIABLE PY_INCLUDE_DIR)
if (NOT _PYTHON_SUCCESS MATCHES 0)
	message(FATAL_ERROR "Python config Error.")
endif()
list(APPEND COMMON_HEADER_DIRS ${PY_INCLUDE_DIR})

# Add LibTorch into COMMON_HEADER_DIRS & COMMON_LIB_DIRS
execute_process(COMMAND ${PYTHON_PATH} "-c" "import os; import torch;
print(os.path.dirname(torch.__file__), end='');"
                  RESULT_VARIABLE _PYTHON_SUCCESS
                  OUTPUT_VARIABLE TORCH_DIR)
if (NOT _PYTHON_SUCCESS MATCHES 0)
	message(FATAL_ERROR "Torch config Error.")
endif()
list(APPEND CMAKE_PREFIX_PATH ${TORCH_DIR})
set(CAFFE2_USE_CUDNN 0)

# [added] Torch's bundled legacy FindCUDA only searches <cuda_root>/include, but conda's
#         CUDA 12/13 packages put headers in <cuda_root>/targets/<arch>/include.
#         Point it at the header directory already found by FindCUDAToolkit.
if (NOT CUDA_TOOLKIT_INCLUDE)
  find_path(_FO_CUDA_RUNTIME_INC cuda_runtime.h HINTS ${CUDAToolkit_INCLUDE_DIRS} NO_DEFAULT_PATH)
  if (_FO_CUDA_RUNTIME_INC)
    set(CUDA_TOOLKIT_INCLUDE ${_FO_CUDA_RUNTIME_INC} CACHE PATH "CUDA toolkit include directory" FORCE)
  endif()
endif()

find_package(Torch REQUIRED)
list(APPEND COMMON_HEADER_DIRS "${TORCH_INCLUDE_DIRS}")
list(APPEND COMMON_LIBS "${TORCH_LIBRARIES}")

# Let COMMON_HEADER_DIRS & COMMON_LIB_DIRS take effect
include_directories(${COMMON_HEADER_DIRS})
link_directories(${COMMON_LIB_DIRS})
link_libraries(${COMMON_LIBS})

# Should turn off CXX11 ABI if pytorch is built with CXX11 ABI off
execute_process(COMMAND ${PYTHON_PATH} "-c" "import torch;
print(torch._C._GLIBCXX_USE_CXX11_ABI,end='');"
                  RESULT_VARIABLE _PYTHON_SUCCESS
                  OUTPUT_VARIABLE USE_CXX11_ABI)
message("-- USE_CXX11_ABI=${USE_CXX11_ABI}")
if (USE_CXX11_ABI)
  set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=1")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=1")
else()
  set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=0")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=0")
endif()

# Let all executable targets go to bin
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# Add subdirectories
add_subdirectory(src)