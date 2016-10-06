#
# CMakeLists.txt  build deltafs and its environment
# 20-Sep-2016  chuck@ece.cmu.edu
#

# command line config:
#
# -DCMAKE_INSTALL_PREFIX=/tmp/delta       # where to install
#

#
# [optional config vars]
# -DCMAKE_PREFIX_PATH='/pkg'              # look for additional installs here
#
# the following also applies for configure scripts:
# -DCMAKE_INCLUDE_PATH='/pkg/include'     # extra include directories
# -DCMAKE_LIBRARY_PATH='/pkg/lib'         # extra library path
#

cmake_minimum_required (VERSION 3.0)
project (deltafs-umbrella NONE)
include (ExternalProject)

# hardwire this on for now
set (CMAKE_BUILD_LIBEV 1)

#
# GenDownloadConfig (result target local-file remote-args ...)
#
# get file locally if present, otherwise download it using remote-args.
#
function (GenDownloadConfig result target local)
    if (EXISTS "${CMAKE_SOURCE_DIR}/cache/${local}")
        message (STATUS "${target}: using cache (${local})")
        # assume correct, but set URL_MD5 to quiet warning
        file (MD5 "${CMAKE_SOURCE_DIR}/cache/${local}" localmd5)
        set (${result} URL "${CMAKE_SOURCE_DIR}/cache/${local}"
                       URL_MD5 ${localmd5} PARENT_SCOPE)
    else ()
        set (${result} ${ARGN} PARENT_SCOPE)
    endif ()
endfunction ()

#
# set up the prefix path for packaged software that we may want to
# link to (e.g. third party libraries).   this will get added to
# the configure command line (for autotools-based projects).
# we want our install prefix to be in the prefix path too (it isn't
# be default).
#
list (APPEND CMAKE_PREFIX_PATH ${CMAKE_INSTALL_PREFIX})
list (REMOVE_DUPLICATES CMAKE_PREFIX_PATH)
foreach (prefix ${CMAKE_PREFIX_PATH})
    list (APPEND CMAKE_INCLUDE_PATH "${prefix}/include")
    list (APPEND CMAKE_LIBRARY_PATH "${prefix}/lib")
endforeach ()
list (REMOVE_DUPLICATES CMAKE_INCLUDE_PATH)
list (REMOVE_DUPLICATES CMAKE_LIBRARY_PATH)

#
# generate a prefix path with an alternate LIST_SEPARATOR for cmake-based
# projects.  needed for ExternalProject because CMAKE_ARGS eats ";"'s.
#
string (REPLACE ";" "^^" CMAKE_QUOTED_PREFIX_PATH "${CMAKE_PREFIX_PATH}")

#
# use above vars to build build CPPFLAGS and LDFLAGS for configure scripts
#
if (CMAKE_INCLUDE_PATH)
  foreach (inc ${CMAKE_INCLUDE_PATH})
      set (cppflags "${cppflags} -I${inc}")
  endforeach ()
  string (SUBSTRING ${cppflags} 1 -1 cppflags)   # remove leading space
  set (cppflags "CPPFLAGS=${cppflags}")
endif ()
if (CMAKE_LIBRARY_PATH)
  foreach (lib ${CMAKE_LIBRARY_PATH})
      set (ldflags "${ldflags} -L${lib}")
  endforeach ()
  string (SUBSTRING ${ldflags} 1 -1 ldflags)
  set (ldflags "LDFLAGS=${ldflags}")
endif ()

#
# now we are ready to start building packages
#

#
# CCI.   note that we enable verbs if we detect the necessary header
# files and libraries.   (note: the find calls below respect the prefix path.)
#
find_library (LIBIBVERBS NAMES ibverbs)
find_path (RDMA_INCLUDE rdma/rdma_cma.h HINTS ${RDMA_INCLUDE_DIR})
if (LIBIBVERBS AND RDMA_INCLUDE)
   set (ibv "--with-verbs")
else ()
   set (ibv "--without-verbs")
endif ()
message (STATUS "CCI ibverbs setting: ${ibv}")

GenDownloadConfig (cci_download cci cci-2.0.tar.gz
    URL http://cci-forum.com/wp-content/uploads/2016/06/cci-2.0.tar.gz
    URL_MD5 070b2ba4eca92a846c093f2cd000d3b2
    TIMEOUT 10
    ### GIT_REPOSITORY "https://github.com/CCI/cci"
    ### GIT_TAG master
)

ExternalProject_Add (cci
    ${cci_download}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure ${cppflags} ${ldflags}
         --prefix=${CMAKE_INSTALL_PREFIX} ${ibv}
    UPDATE_COMMAND ""
)

ExternalProject_Add_Step(cci prepare
    COMMAND ${CMAKE_SOURCE_DIR}/scripts/ensure-autogen <SOURCE_DIR>/autogen.pl
    COMMENT "preparing source for configure"
    DEPENDEES update
    DEPENDERS configure
    WORKING_DIRECTORY <SOURCE_DIR>
)

#
# BMI
#
set (bmipatch "${CMAKE_SOURCE_DIR}/patches/bmi-${CMAKE_SYSTEM_NAME}-patch")
if (EXISTS ${bmipatch})
    set (bmipatchcmd patch -i ${bmipatch} -d <SOURCE_DIR>)
endif ()

GenDownloadConfig (bmi_download bmi bmi.tar.gz
    GIT_REPOSITORY "http://git.mcs.anl.gov/bmi.git"
    ### GIT_TAG master
)

ExternalProject_Add (bmi
    ${bmi_download}
    PATCH_COMMAND ${bmipatchcmd}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=${CMAKE_INSTALL_PREFIX}
                      --enable-shared --enable-bmi-only
    UPDATE_COMMAND ""
)

ExternalProject_Add_Step(bmi prepare
    COMMAND ${CMAKE_SOURCE_DIR}/scripts/ensure-autogen <SOURCE_DIR>/prepare
    COMMENT "preparing source for configure"
    DEPENDEES update
    DEPENDERS configure
    WORKING_DIRECTORY <SOURCE_DIR>
)

#
# mercury
#
# XXXCDC: bmi always installs under .so, cci uses ${suf} (below)
set (suf "${CMAKE_SHARED_LIBRARY_SUFFIX}")  # ".so" ".dylib"

GenDownloadConfig (mercury_download mercury mercury.tar.gz
    GIT_REPOSITORY "https://github.com/mercury-hpc/mercury.git"
    ### GIT_TAG master
)

ExternalProject_Add (mercury
    DEPENDS cci bmi
    LIST_SEPARATOR ^^                    # for passing CMAKE_PREFIX_PATH
    ${mercury_download}
    # GIT_TAG master
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
               -DCMAKE_PREFIX_PATH=${CMAKE_QUOTED_PREFIX_PATH}
               -DCMAKE_EXPORT_NO_PACKAGE_REGISTRY=1
               -DNA_USE_MPI=OFF -DNA_USE_CCI=ON -DNA_USE_BMI=ON
               -DBUILD_SHARED_LIBS=ON -DBUILD_TESTING=ON
               -DMERCURY_USE_BOOST_PP=ON -DMERCURY_USE_CHECKSUMS:BOOL=OFF
               -DNA_CCI_USE_POLL:BOOL=ON
               -DBMI_INCLUDE_DIR=${CMAKE_INSTALL_PREFIX}/include
               -DBMI_LIBRARY=${CMAKE_INSTALL_PREFIX}/lib/libbmi.so
               -DCCI_INCLUDE_DIR=${CMAKE_INSTALL_PREFIX}/include
               -DCCI_LIBRARY=${CMAKE_INSTALL_PREFIX}/lib/libcci${suf}
    UPDATE_COMMAND ""

# XXX: not all tests run, so no TEST_COMMAND for now
#ctest -E 'mercury_bulk_cci_tcp|mercury_bulk_seg_cci_tcp|mercury_posix_cci_tcp'
# some issues with cci/tcp transport

)

#
# Snappy (XXX: using tar file, having aclocal issues on nerc)
#
GenDownloadConfig (snappy_download snappy snappy-1.1.3.tar.gz
    URL https://github.com/google/snappy/releases/download/1.1.3/snappy-1.1.3.tar.gz
    URL_MD5 7358c82f133dc77798e4c2062a749b73
    TIMEOUT 10
    ### GIT_REPOSITORY "https://github.com/google/snappy"
    ### GIT_TAG master
)

ExternalProject_Add (snappy
    ${snappy_download}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure ${cppflags} ${ldflags}
         --prefix=${CMAKE_INSTALL_PREFIX}
    UPDATE_COMMAND ""
)

ExternalProject_Add_Step(snappy prepare
    COMMAND ${CMAKE_SOURCE_DIR}/scripts/ensure-autogen <SOURCE_DIR>/autogen.sh
    COMMENT "preparing source for configure"
    DEPENDEES update
    DEPENDERS configure
    WORKING_DIRECTORY <SOURCE_DIR>
)

#
# gflags
#
GenDownloadConfig (gflags_download gflags gflags.tar.gz
    GIT_REPOSITORY "https://github.com/gflags/gflags"
    ### GIT_TAG master
)

ExternalProject_Add (gflags
    LIST_SEPARATOR ^^                    # for passing CMAKE_PREFIX_PATH
    ${gflags_download}
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
               -DCMAKE_PREFIX_PATH=${CMAKE_QUOTED_PREFIX_PATH}
               -DCMAKE_EXPORT_NO_PACKAGE_REGISTRY=1
               -DBUILD_SHARED_LIBS=ON
    UPDATE_COMMAND ""
)

#
# glog
#
GenDownloadConfig (glog_download glog glog.tar.gz
    GIT_REPOSITORY "https://github.com/google/glog"
    ### GIT_TAG master
)

ExternalProject_Add (glog
    DEPENDS gflags
    LIST_SEPARATOR ^^                    # for passing CMAKE_PREFIX_PATH
    ${glog_download}
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
               -DCMAKE_PREFIX_PATH=${CMAKE_QUOTED_PREFIX_PATH}
               -DCMAKE_EXPORT_NO_PACKAGE_REGISTRY=1
               -DBUILD_SHARED_LIBS=ON
    UPDATE_COMMAND ""
)

#
# nvml
#
set (nvmlpatch "${CMAKE_SOURCE_DIR}/patches/nvml-patch")
if (EXISTS ${nvmlpatch})
    set (nvmlpatchcmd patch -p1 -i ${nvmlpatch} -d <SOURCE_DIR>)
endif ()

GenDownloadConfig (nvml_download nvml nvml.tar.gz
    GIT_REPOSITORY "https://github.com/pmem/nvml"
    ### GIT_TAG master
)

ExternalProject_Add (nvml
    ${nvml_download}
    PATCH_COMMAND ${nvmlpatchcmd}
    CONFIGURE_COMMAND ""
    BUILD_IN_SOURCE 1
    BUILD_COMMAND make prefix=${CMAKE_INSTALL_PREFIX}
        COMMAND make prefix=${CMAKE_INSTALL_PREFIX} test
    INSTALL_COMMAND make prefix=${CMAKE_INSTALL_PREFIX} install
    # TEST_COMMAND make prefix=${CMAKE_INSTALL_PREFIX} check
    UPDATE_COMMAND ""
)

#
# argobots
#
set (argopatch 
    "${CMAKE_SOURCE_DIR}/patches/argobots-${CMAKE_SYSTEM_NAME}-patch")
if (EXISTS ${argopatch})
    set (argopatchcmd patch -p1 -i ${argopatch} -d <SOURCE_DIR>)
endif ()

GenDownloadConfig (argobots_download argobots argobots.tar.gz
    GIT_REPOSITORY "http://git.mcs.anl.gov/argo/argobots.git"
    ### GIT_TAG master
)

ExternalProject_Add (argobots
    ${argobots_download}
    PATCH_COMMAND ${argopatchcmd}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=${CMAKE_INSTALL_PREFIX}
    TEST_COMMAND make -C test check
    UPDATE_COMMAND ""
)

ExternalProject_Add_Step(argobots prepare
    COMMAND ${CMAKE_SOURCE_DIR}/scripts/ensure-autogen <SOURCE_DIR>/autogen.sh
    COMMENT "preparing source for configure"
    DEPENDEES update
    DEPENDERS configure
    WORKING_DIRECTORY <SOURCE_DIR>
)

if (CMAKE_BUILD_LIBEV)
#
# libev
#
GenDownloadConfig (libev_download libev libev-4.22.tar.gz
    URL http://dist.schmorp.de/libev/libev-4.22.tar.gz
    URL_MD5 bf0007e37240103d3c2be80ca9bee3f9
    ### GIT_TAG master
)
ExternalProject_Add(libev
    ${libev_download}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure ${cppflags} ${ldflags}
       --prefix=${CMAKE_INSTALL_PREFIX}
)
endif ()

#
# abt-snoozer
#

# configured based on which libev we are using
if (CMAKE_BUILD_LIBEV)
    set (abt-snoozer-extra-dep libev)
    set (EVHDIR ${CMAKE_INSTALL_PREFIX})
else ()
    find_path (EVH ev.h)
    if (NOT EVH)
        message (FATAL_ERROR "Can't find libev eh.h")
    endif ()
    get_filename_component (EVHDIR ${EVH} DIRECTORY)   # don't want "include"
endif ()

set (abtspatch
    "${CMAKE_SOURCE_DIR}/patches/abt-snoozer-${CMAKE_SYSTEM_NAME}-patch")
if (EXISTS ${abtspatch})
    set (abtspatchcmd patch -i ${abtspatch} -d <SOURCE_DIR>)
endif ()

GenDownloadConfig (abts_download abt-snoozer abt-snoozer.tar.gz
    GIT_REPOSITORY "https://xgitlab.cels.anl.gov/sds/abt-snoozer.git"
    ### GIT_TAG master
)

ExternalProject_Add (abt-snoozer
    DEPENDS argobots ${abt-snoozer-extra-dep}
    ${abts_download}
    PATCH_COMMAND ${abtspatchcmd}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=${CMAKE_INSTALL_PREFIX}
        --with-libev=${EVHDIR} --enable-shared
        PKG_CONFIG_PATH=${CMAKE_INSTALL_PREFIX}/lib/pkgconfig
    TEST_COMMAND make check
    UPDATE_COMMAND ""
)

ExternalProject_Add_Step(abt-snoozer prepare
    COMMAND ${CMAKE_SOURCE_DIR}/scripts/ensure-autogen <SOURCE_DIR>/prepare.sh
    COMMENT "preparing source for configure"
    DEPENDEES update
    DEPENDERS configure
    WORKING_DIRECTORY <SOURCE_DIR>
)

#
# margo
#
GenDownloadConfig (margo_download margo margo.tar.gz
    GIT_REPOSITORY "https://xgitlab.cels.anl.gov/sds/margo.git"
    ### GIT_TAG master
)

ExternalProject_Add(margo
    DEPENDS abt-snoozer mercury
    ${margo_download}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=${CMAKE_INSTALL_PREFIX}
        PKG_CONFIG_PATH=${CMAKE_INSTALL_PREFIX}/lib/pkgconfig
    # TEST_COMMAND make check     # two currently failing
    UPDATE_COMMAND ""
)

ExternalProject_Add_Step(margo prepare
    COMMAND ${CMAKE_SOURCE_DIR}/scripts/ensure-autogen <SOURCE_DIR>/prepare.sh
    COMMENT "preparing source for configure"
    DEPENDEES update
    DEPENDERS configure
    WORKING_DIRECTORY <SOURCE_DIR>
)

#
# bake-bulk
#
GenDownloadConfig (bake_download bake-bulk bake-bulk.tar.gz
    GIT_REPOSITORY "git@xgitlab.cels.anl.gov:sds/bake-bulk.git"
    ### GIT_TAG master
)

ExternalProject_Add(bake-bulk
    DEPENDS margo mercury nvml
    ${bake_download}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=${CMAKE_INSTALL_PREFIX}
        PKG_CONFIG_PATH=${CMAKE_INSTALL_PREFIX}/lib/pkgconfig
    UPDATE_COMMAND ""
)

ExternalProject_Add_Step(bake-bulk prepare
    COMMAND ${CMAKE_SOURCE_DIR}/scripts/ensure-autogen <SOURCE_DIR>/prepare.sh
    COMMENT "preparing source for configure"
    DEPENDEES update
    DEPENDERS configure
    WORKING_DIRECTORY <SOURCE_DIR>
)

#
# mpich
#
GenDownloadConfig (mpich_download mpich mpich-3.2.tar.gz
    URL http://www.mpich.org/static/downloads/3.2/mpich-3.2.tar.gz
    URL_MD5 f414cfa77099cd1fa1a5ae4e22db508a
    ### GIT_TAG master
)
ExternalProject_Add(mpich
    ${mpich_download}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure ${cppflags} ${ldflags}
       --disable-fortran
       --prefix=${CMAKE_INSTALL_PREFIX}
)

# only do mpich if we directly asked for it
set_property (TARGET mpich PROPERTY EXCLUDE_FROM_ALL True)

#
# automake
#
GenDownloadConfig (automake_download automake automake-1.15.tar.gz
    URL http://ftp.gnu.org/gnu/automake/automake-1.15.tar.gz
    URL_MD5 716946a105ca228ab545fc37a70df3a3
    ### GIT_TAG master
)
ExternalProject_Add(automake
    ${automake_download}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure ${cppflags} ${ldflags}
       --prefix=${CMAKE_INSTALL_PREFIX}
)

# only do automake if we directly asked for it (mainly for travis)
set_property (TARGET automake PROPERTY EXCLUDE_FROM_ALL True)
