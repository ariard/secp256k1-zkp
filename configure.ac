AC_PREREQ([2.60])

# The package (a.k.a. release) version is based on semantic versioning 2.0.0 of
# the API. All changes in experimental modules are treated as
# backwards-compatible and therefore at most increase the minor version.
define(_PKG_VERSION_MAJOR, 0)
define(_PKG_VERSION_MINOR, 1)
define(_PKG_VERSION_BUILD, 0)
define(_PKG_VERSION_IS_RELEASE, false)

# The library version is based on libtool versioning of the ABI. The set of
# rules for updating the version can be found here:
# https://www.gnu.org/software/libtool/manual/html_node/Updating-version-info.html
# All changes in experimental modules are treated as if they don't affect the
# interface and therefore only increase the revision.
define(_LIB_VERSION_CURRENT, 0)
define(_LIB_VERSION_REVISION, 0)
define(_LIB_VERSION_AGE, 0)

AC_INIT([libsecp256k1],m4_join([.], _PKG_VERSION_MAJOR, _PKG_VERSION_MINOR, _PKG_VERSION_BUILD)m4_if(_PKG_VERSION_IS_RELEASE, [true], [], [-pre]),[https://github.com/bitcoin-core/secp256k1/issues],[libsecp256k1],[https://github.com/bitcoin-core/secp256k1])

AC_CONFIG_AUX_DIR([build-aux])
AC_CONFIG_MACRO_DIR([build-aux/m4])
AC_CANONICAL_HOST
AH_TOP([#ifndef LIBSECP256K1_CONFIG_H])
AH_TOP([#define LIBSECP256K1_CONFIG_H])
AH_BOTTOM([#endif /*LIBSECP256K1_CONFIG_H*/])

# Require Automake 1.11.2 for AM_PROG_AR
AM_INIT_AUTOMAKE([1.11.2 foreign subdir-objects])

# Make the compilation flags quiet unless V=1 is used.
m4_ifdef([AM_SILENT_RULES], [AM_SILENT_RULES([yes])])

AC_PROG_CC
if test x"$ac_cv_prog_cc_c89" = x"no"; then
  AC_MSG_ERROR([c89 compiler support required])
fi
AM_PROG_AS
AM_PROG_AR

LT_INIT([win32-dll])

build_windows=no

case $host_os in
  *darwin*)
     if  test x$cross_compiling != xyes; then
       AC_CHECK_PROG([BREW], brew, brew)
       if test x$BREW = xbrew; then
         # These Homebrew packages may be keg-only, meaning that they won't be found
         # in expected paths because they may conflict with system files. Ask
         # Homebrew where each one is located, then adjust paths accordingly.
         if $BREW list --versions valgrind >/dev/null; then
           valgrind_prefix=$($BREW --prefix valgrind 2>/dev/null)
           VALGRIND_CPPFLAGS="-I$valgrind_prefix/include"
         fi
       else
         AC_CHECK_PROG([PORT], port, port)
         # If homebrew isn't installed and macports is, add the macports default paths
         # as a last resort.
         if test x$PORT = xport; then
           CPPFLAGS="$CPPFLAGS -isystem /opt/local/include"
           LDFLAGS="$LDFLAGS -L/opt/local/lib"
         fi
       fi
     fi
   ;;
   cygwin*|mingw*)
     build_windows=yes
   ;;
esac

# Try if some desirable compiler flags are supported and append them to SECP_CFLAGS.
#
# These are our own flags, so we append them to our own SECP_CFLAGS variable (instead of CFLAGS) as
# recommended in the automake manual (Section "Flag Variables Ordering"). CFLAGS belongs to the user
# and we are not supposed to touch it. In the Makefile, we will need to ensure that SECP_CFLAGS
# is prepended to CFLAGS when invoking the compiler so that the user always has the last word (flag).
#
# Another advantage of not touching CFLAGS is that the contents of CFLAGS will be picked up by
# libtool for compiling helper executables. For example, when compiling for Windows, libtool will
# generate entire wrapper executables (instead of simple wrapper scripts as on Unix) to ensure
# proper operation of uninstalled programs linked by libtool against the uninstalled shared library.
# These executables are compiled from C source file for which our flags may not be appropriate,
# e.g., -std=c89 flag has lead to undesirable warnings in the past.
#
# TODO We should analogously not touch CPPFLAGS and LDFLAGS but currently there are no issues.
AC_DEFUN([SECP_TRY_APPEND_DEFAULT_CFLAGS], [
    # Try to append -Werror=unknown-warning-option to CFLAGS temporarily. Otherwise clang will
    # not error out if it gets unknown warning flags and the checks here will always succeed
    # no matter if clang knows the flag or not.
    SECP_TRY_APPEND_DEFAULT_CFLAGS_saved_CFLAGS="$CFLAGS"
    SECP_TRY_APPEND_CFLAGS([-Werror=unknown-warning-option], CFLAGS)

    SECP_TRY_APPEND_CFLAGS([-std=c89 -pedantic -Wno-long-long -Wnested-externs -Wshadow -Wstrict-prototypes -Wundef], $1) # GCC >= 3.0, -Wlong-long is implied by -pedantic.
    SECP_TRY_APPEND_CFLAGS([-Wno-overlength-strings], $1) # GCC >= 4.2, -Woverlength-strings is implied by -pedantic.
    SECP_TRY_APPEND_CFLAGS([-Wall], $1) # GCC >= 2.95 and probably many other compilers
    SECP_TRY_APPEND_CFLAGS([-Wno-unused-function], $1) # GCC >= 3.0, -Wunused-function is implied by -Wall.
    SECP_TRY_APPEND_CFLAGS([-Wextra], $1) # GCC >= 3.4, this is the newer name of -W, which we don't use because older GCCs will warn about unused functions.
    SECP_TRY_APPEND_CFLAGS([-Wcast-align], $1) # GCC >= 2.95
    SECP_TRY_APPEND_CFLAGS([-Wcast-align=strict], $1) # GCC >= 8.0
    SECP_TRY_APPEND_CFLAGS([-Wconditional-uninitialized], $1) # Clang >= 3.0 only
    SECP_TRY_APPEND_CFLAGS([-fvisibility=hidden], $1) # GCC >= 4.0

    CFLAGS="$SECP_TRY_APPEND_DEFAULT_CFLAGS_saved_CFLAGS"
])
SECP_TRY_APPEND_DEFAULT_CFLAGS(SECP_CFLAGS)

###
### Define config arguments
###

# In dev mode, we enable all binaries and modules by default but individual options can still be overridden explicitly.
# Check for dev mode first because SECP_SET_DEFAULT needs enable_dev_mode set.
AC_ARG_ENABLE(dev_mode, [], [],
    [enable_dev_mode=no])

AC_ARG_ENABLE(benchmark,
    AS_HELP_STRING([--enable-benchmark],[compile benchmark [default=yes]]), [],
    [SECP_SET_DEFAULT([enable_benchmark], [yes], [yes])])

AC_ARG_ENABLE(coverage,
    AS_HELP_STRING([--enable-coverage],[enable compiler flags to support kcov coverage analysis [default=no]]), [],
    [SECP_SET_DEFAULT([enable_coverage], [no], [no])])

AC_ARG_ENABLE(tests,
    AS_HELP_STRING([--enable-tests],[compile tests [default=yes]]), [],
    [SECP_SET_DEFAULT([enable_tests], [yes], [yes])])

AC_ARG_ENABLE(experimental,
    AS_HELP_STRING([--enable-experimental],[allow experimental configure options [default=no]]), [],
    [SECP_SET_DEFAULT([enable_experimental], [no], [yes])])

AC_ARG_ENABLE(exhaustive_tests,
    AS_HELP_STRING([--enable-exhaustive-tests],[compile exhaustive tests [default=yes]]), [],
    [SECP_SET_DEFAULT([enable_exhaustive_tests], [yes], [yes])])

AC_ARG_ENABLE(examples,
    AS_HELP_STRING([--enable-examples],[compile the examples [default=no]]), [],
    [SECP_SET_DEFAULT([enable_examples], [no], [yes])])

AC_ARG_ENABLE(module_ecdh,
    AS_HELP_STRING([--enable-module-ecdh],[enable ECDH module [default=no]]), [],
    [SECP_SET_DEFAULT([enable_module_ecdh], [no], [yes])])

AC_ARG_ENABLE(module_musig,
    AS_HELP_STRING([--enable-module-musig],[enable MuSig module (experimental)]),
    [],
    [SECP_SET_DEFAULT([enable_module_musig], [no], [yes])])

AC_ARG_ENABLE(module_recovery,
    AS_HELP_STRING([--enable-module-recovery],[enable ECDSA pubkey recovery module [default=no]]), [],
    [SECP_SET_DEFAULT([enable_module_recovery], [no], [yes])])

AC_ARG_ENABLE(module_generator,
    AS_HELP_STRING([--enable-module-generator],[enable NUMS generator module [default=no]]),
    [],
    [SECP_SET_DEFAULT([enable_module_generator], [no], [yes])])

AC_ARG_ENABLE(module_rangeproof,
    AS_HELP_STRING([--enable-module-rangeproof],[enable Pedersen / zero-knowledge range proofs module [default=no]]),
    [],
    [SECP_SET_DEFAULT([enable_module_rangeproof], [no], [yes])])

AC_ARG_ENABLE(module_whitelist,
    AS_HELP_STRING([--enable-module-whitelist],[enable key whitelisting module [default=no]]),
    [],
    [SECP_SET_DEFAULT([enable_module_whitelist], [no], [yes])])

AC_ARG_ENABLE(module_extrakeys,
    AS_HELP_STRING([--enable-module-extrakeys],[enable extrakeys module [default=no]]), [],
    [SECP_SET_DEFAULT([enable_module_extrakeys], [no], [yes])])

AC_ARG_ENABLE(module_schnorrsig,
    AS_HELP_STRING([--enable-module-schnorrsig],[enable schnorrsig module [default=no]]), [],
    [SECP_SET_DEFAULT([enable_module_schnorrsig], [no], [yes])])

AC_ARG_ENABLE(module_ecdsa_s2c,
    AS_HELP_STRING([--enable-module-ecdsa-s2c],[enable ECDSA sign-to-contract module [default=no]]),
    [],
    [SECP_SET_DEFAULT([enable_module_ecdsa_s2c], [no], [yes])])

AC_ARG_ENABLE(module_ecdsa-adaptor,
    AS_HELP_STRING([--enable-module-ecdsa-adaptor],[enable ECDSA adaptor module [default=no]]),
    [],
    [SECP_SET_DEFAULT([enable_module_ecdsa_adaptor], [no], [yes])])

AC_ARG_ENABLE(external_default_callbacks,
    AS_HELP_STRING([--enable-external-default-callbacks],[enable external default callback functions [default=no]]), [],
    [SECP_SET_DEFAULT([enable_external_default_callbacks], [no], [no])])

AC_ARG_ENABLE(module_surjectionproof,
    AS_HELP_STRING([--enable-module-surjectionproof],[enable surjection proof module [default=no]]),
    [],
    [SECP_SET_DEFAULT([enable_module_surjectionproof], [no], [yes])])

AC_ARG_ENABLE(reduced_surjection_proof_size,
    AS_HELP_STRING([--enable-reduced-surjection-proof-size],[use reduced surjection proof size (disabling parsing and verification) [default=no]]),
    [],
    [SECP_SET_DEFAULT([use_reduced_surjection_proof_size], [no], [no])])

# Test-only override of the (autodetected by the C code) "widemul" setting.
# Legal values are int64 (for [u]int64_t), int128 (for [unsigned] __int128), and auto (the default).
AC_ARG_WITH([test-override-wide-multiply], [] ,[set_widemul=$withval], [set_widemul=auto])

AC_ARG_WITH([asm], [AS_HELP_STRING([--with-asm=x86_64|arm|no|auto],
[assembly optimizations to use (experimental: arm) [default=auto]])],[req_asm=$withval], [req_asm=auto])

AC_ARG_WITH([ecmult-window], [AS_HELP_STRING([--with-ecmult-window=SIZE|auto],
[window size for ecmult precomputation for verification, specified as integer in range [2..24].]
[Larger values result in possibly better performance at the cost of an exponentially larger precomputed table.]
[The table will store 2^(SIZE-1) * 64 bytes of data but can be larger in memory due to platform-specific padding and alignment.]
[A window size larger than 15 will require you delete the prebuilt precomputed_ecmult.c file so that it can be rebuilt.]
[For very large window sizes, use "make -j 1" to reduce memory use during compilation.]
["auto" is a reasonable setting for desktop machines (currently 15). [default=auto]]
)],
[req_ecmult_window=$withval], [req_ecmult_window=auto])

AC_ARG_WITH([ecmult-gen-precision], [AS_HELP_STRING([--with-ecmult-gen-precision=2|4|8|auto],
[Precision bits to tune the precomputed table size for signing.]
[The size of the table is 32kB for 2 bits, 64kB for 4 bits, 512kB for 8 bits of precision.]
[A larger table size usually results in possible faster signing.]
["auto" is a reasonable setting for desktop machines (currently 4). [default=auto]]
)],
[req_ecmult_gen_precision=$withval], [req_ecmult_gen_precision=auto])

AC_ARG_WITH([valgrind], [AS_HELP_STRING([--with-valgrind=yes|no|auto],
[Build with extra checks for running inside Valgrind [default=auto]]
)],
[req_valgrind=$withval], [req_valgrind=auto])

###
### Handle config options (except for modules)
###

if test x"$req_valgrind" = x"no"; then
  enable_valgrind=no
else
  SECP_VALGRIND_CHECK
  if test x"$has_valgrind" != x"yes"; then
    if test x"$req_valgrind" = x"yes"; then
      AC_MSG_ERROR([Valgrind support explicitly requested but valgrind/memcheck.h header not available])
    fi
    enable_valgrind=no
  else
    enable_valgrind=yes
  fi
fi
AM_CONDITIONAL([VALGRIND_ENABLED],[test "$enable_valgrind" = "yes"])

if test x"$enable_coverage" = x"yes"; then
    AC_DEFINE(COVERAGE, 1, [Define this symbol to compile out all VERIFY code])
    SECP_CFLAGS="-O0 --coverage $SECP_CFLAGS"
    LDFLAGS="--coverage $LDFLAGS"
else
    # Most likely the CFLAGS already contain -O2 because that is autoconf's default.
    # We still add it here because passing it twice is not an issue, and handling
    # this case would just add unnecessary complexity (see #896).
    SECP_CFLAGS="-O2 $SECP_CFLAGS"
fi

AC_MSG_CHECKING([for __builtin_popcount])
AC_LINK_IFELSE([AC_LANG_SOURCE([[void myfunc() {__builtin_popcount(0);}]])],
    [ AC_MSG_RESULT([yes]);AC_DEFINE(HAVE_BUILTIN_POPCOUNT,1,[Define this symbol if __builtin_popcount is available]) ],
    [ AC_MSG_RESULT([no])
    ])

AC_MSG_CHECKING([for  __builtin_clzll])
AC_LINK_IFELSE([AC_LANG_SOURCE([[void myfunc() { __builtin_clzll(1);}]])],
    [ AC_MSG_RESULT([yes]);AC_DEFINE(HAVE_BUILTIN_CLZLL,1,[Define this symbol if  __builtin_clzll is available]) ],
    [ AC_MSG_RESULT([no])
    ])

if test x"$req_asm" = x"auto"; then
  SECP_64BIT_ASM_CHECK
  if test x"$has_64bit_asm" = x"yes"; then
    set_asm=x86_64
  fi
  if test x"$set_asm" = x; then
    set_asm=no
  fi
else
  set_asm=$req_asm
  case $set_asm in
  x86_64)
    SECP_64BIT_ASM_CHECK
    if test x"$has_64bit_asm" != x"yes"; then
      AC_MSG_ERROR([x86_64 assembly optimization requested but not available])
    fi
    ;;
  arm)
    ;;
  no)
    ;;
  *)
    AC_MSG_ERROR([invalid assembly optimization selection])
    ;;
  esac
fi

# Select assembly optimization
enable_external_asm=no

case $set_asm in
x86_64)
  AC_DEFINE(USE_ASM_X86_64, 1, [Define this symbol to enable x86_64 assembly optimizations])
  ;;
arm)
  enable_external_asm=yes
  ;;
no)
  ;;
*)
  AC_MSG_ERROR([invalid assembly optimizations])
  ;;
esac

if test x"$enable_external_asm" = x"yes"; then
  AC_DEFINE(USE_EXTERNAL_ASM, 1, [Define this symbol if an external (non-inline) assembly implementation is used])
fi


# Select wide multiplication implementation
case $set_widemul in
int128)
  AC_DEFINE(USE_FORCE_WIDEMUL_INT128, 1, [Define this symbol to force the use of the (unsigned) __int128 based wide multiplication implementation])
  ;;
int64)
  AC_DEFINE(USE_FORCE_WIDEMUL_INT64, 1, [Define this symbol to force the use of the (u)int64_t based wide multiplication implementation])
  ;;
auto)
  ;;
*)
  AC_MSG_ERROR([invalid wide multiplication implementation])
  ;;
esac

# Set ecmult window size
if test x"$req_ecmult_window" = x"auto"; then
  set_ecmult_window=15
else
  set_ecmult_window=$req_ecmult_window
fi

error_window_size=['window size for ecmult precomputation not an integer in range [2..24] or "auto"']
case $set_ecmult_window in
''|*[[!0-9]]*)
  # no valid integer
  AC_MSG_ERROR($error_window_size)
  ;;
*)
  if test "$set_ecmult_window" -lt 2 -o "$set_ecmult_window" -gt 24 ; then
    # not in range
    AC_MSG_ERROR($error_window_size)
  fi
  AC_DEFINE_UNQUOTED(ECMULT_WINDOW_SIZE, $set_ecmult_window, [Set window size for ecmult precomputation])
  ;;
esac

# Set ecmult gen precision
if test x"$req_ecmult_gen_precision" = x"auto"; then
  set_ecmult_gen_precision=4
else
  set_ecmult_gen_precision=$req_ecmult_gen_precision
fi

case $set_ecmult_gen_precision in
2|4|8)
  AC_DEFINE_UNQUOTED(ECMULT_GEN_PREC_BITS, $set_ecmult_gen_precision, [Set ecmult gen precision bits])
  ;;
*)
  AC_MSG_ERROR(['ecmult gen precision not 2, 4, 8 or "auto"'])
  ;;
esac

if test x"$enable_valgrind" = x"yes"; then
  SECP_INCLUDES="$SECP_INCLUDES $VALGRIND_CPPFLAGS"
fi

# Add -Werror and similar flags passed from the outside (for testing, e.g., in CI)
SECP_CFLAGS="$SECP_CFLAGS $WERROR_CFLAGS"

###
### Handle module options
###

# Besides testing whether modules are enabled, the following code also enables
# module dependencies. The order of the tests matters: the dependency must be
# tested first.

if test x"$enable_module_ecdh" = x"yes"; then
  AC_DEFINE(ENABLE_MODULE_ECDH, 1, [Define this symbol to enable the ECDH module])
fi

if test x"$enable_module_musig" = x"yes"; then
  AC_DEFINE(ENABLE_MODULE_MUSIG, 1, [Define this symbol to enable the MuSig module])
  enable_module_schnorrsig=yes
fi

if test x"$enable_module_recovery" = x"yes"; then
  AC_DEFINE(ENABLE_MODULE_RECOVERY, 1, [Define this symbol to enable the ECDSA pubkey recovery module])
fi

if test x"$enable_module_whitelist" = x"yes"; then
  enable_module_rangeproof=yes
  AC_DEFINE(ENABLE_MODULE_WHITELIST, 1, [Define this symbol to enable the key whitelisting module])
fi

if test x"$enable_module_surjectionproof" = x"yes"; then
  enable_module_rangeproof=yes
  AC_DEFINE(ENABLE_MODULE_SURJECTIONPROOF, 1, [Define this symbol to enable the surjection proof module])
fi

if test x"$enable_module_rangeproof" = x"yes"; then
  enable_module_generator=yes
  AC_DEFINE(ENABLE_MODULE_RANGEPROOF, 1, [Define this symbol to enable the Pedersen / zero knowledge range proof module])
fi

if test x"$enable_module_generator" = x"yes"; then
  AC_DEFINE(ENABLE_MODULE_GENERATOR, 1, [Define this symbol to enable the NUMS generator module])
fi

if test x"$enable_module_schnorrsig" = x"yes"; then
  AC_DEFINE(ENABLE_MODULE_SCHNORRSIG, 1, [Define this symbol to enable the schnorrsig module])
  enable_module_extrakeys=yes
fi

if test x"$enable_module_extrakeys" = x"yes"; then
  AC_DEFINE(ENABLE_MODULE_EXTRAKEYS, 1, [Define this symbol to enable the extrakeys module])
fi

if test x"$enable_module_ecdsa_s2c" = x"yes"; then
  AC_DEFINE(ENABLE_MODULE_ECDSA_S2C, 1, [Define this symbol to enable the ECDSA sign-to-contract module])
fi

if test x"$enable_external_default_callbacks" = x"yes"; then
  AC_DEFINE(USE_EXTERNAL_DEFAULT_CALLBACKS, 1, [Define this symbol if an external implementation of the default callbacks is used])
fi

if test x"$use_reduced_surjection_proof_size" = x"yes"; then
  AC_DEFINE(USE_REDUCED_SURJECTION_PROOF_SIZE, 1, [Define this symbol to reduce SECP256K1_SURJECTIONPROOF_MAX_N_INPUTS to 16, disabling parsing and verification])
fi

if test x"$enable_module_ecdsa_adaptor" = x"yes"; then
  AC_DEFINE(ENABLE_MODULE_ECDSA_ADAPTOR, 1, [Define this symbol to enable the ECDSA adaptor module])
fi

###
### Check for --enable-experimental if necessary
###

if test x"$enable_experimental" = x"yes"; then
  AC_MSG_NOTICE([******])
  AC_MSG_NOTICE([WARNING: experimental build])
  AC_MSG_NOTICE([Experimental features do not have stable APIs or properties, and may not be safe for production use.])
  AC_MSG_NOTICE([******])
else
  # The order of the following tests matters. If the user enables a dependent
  # module (which automatically enables the module dependencies) we want to
  # print an error for the dependent module, not the module dependency. Hence,
  # we first test dependent modules.
  if test x"$enable_module_whitelist" = x"yes"; then
    AC_MSG_ERROR([Key whitelisting module is experimental. Use --enable-experimental to allow.])
  fi
  if test x"$enable_module_surjectionproof" = x"yes"; then
    AC_MSG_ERROR([Surjection proof module is experimental. Use --enable-experimental to allow.])
  fi
  if test x"$enable_module_rangeproof" = x"yes"; then
    AC_MSG_ERROR([Range proof module is experimental. Use --enable-experimental to allow.])
  fi
  if test x"$enable_module_generator" = x"yes"; then
    AC_MSG_ERROR([NUMS generator module is experimental. Use --enable-experimental to allow.])
  fi
  if test x"$enable_module_musig" = x"yes"; then
    AC_MSG_ERROR([MuSig module is experimental. Use --enable-experimental to allow.])
  fi
  if test x"$enable_module_ecdsa_s2c" = x"yes"; then
    AC_MSG_ERROR([ECDSA sign-to-contract module module is experimental. Use --enable-experimental to allow.])
  fi
  if test x"$enable_module_ecdsa_adaptor" = x"yes"; then
    AC_MSG_ERROR([ecdsa adaptor signatures module is experimental. Use --enable-experimental to allow.])
  fi
  if test x"$set_asm" = x"arm"; then
    AC_MSG_ERROR([ARM assembly optimization is experimental. Use --enable-experimental to allow.])
  fi
fi

###
### Generate output
###

AC_CONFIG_HEADERS([src/libsecp256k1-config.h])
AC_CONFIG_FILES([Makefile libsecp256k1.pc])
AC_SUBST(SECP_INCLUDES)
AC_SUBST(SECP_LIBS)
AC_SUBST(SECP_TEST_LIBS)
AC_SUBST(SECP_TEST_INCLUDES)
AC_SUBST(SECP_CFLAGS)
AM_CONDITIONAL([ENABLE_COVERAGE], [test x"$enable_coverage" = x"yes"])
AM_CONDITIONAL([USE_TESTS], [test x"$enable_tests" != x"no"])
AM_CONDITIONAL([USE_EXHAUSTIVE_TESTS], [test x"$enable_exhaustive_tests" != x"no"])
AM_CONDITIONAL([USE_EXAMPLES], [test x"$enable_examples" != x"no"])
AM_CONDITIONAL([USE_BENCHMARK], [test x"$enable_benchmark" = x"yes"])
AM_CONDITIONAL([ENABLE_MODULE_ECDH], [test x"$enable_module_ecdh" = x"yes"])
AM_CONDITIONAL([ENABLE_MODULE_MUSIG], [test x"$enable_module_musig" = x"yes"])
AM_CONDITIONAL([ENABLE_MODULE_RECOVERY], [test x"$enable_module_recovery" = x"yes"])
AM_CONDITIONAL([ENABLE_MODULE_GENERATOR], [test x"$enable_module_generator" = x"yes"])
AM_CONDITIONAL([ENABLE_MODULE_RANGEPROOF], [test x"$enable_module_rangeproof" = x"yes"])
AM_CONDITIONAL([ENABLE_MODULE_WHITELIST], [test x"$enable_module_whitelist" = x"yes"])
AM_CONDITIONAL([ENABLE_MODULE_EXTRAKEYS], [test x"$enable_module_extrakeys" = x"yes"])
AM_CONDITIONAL([ENABLE_MODULE_SCHNORRSIG], [test x"$enable_module_schnorrsig" = x"yes"])
AM_CONDITIONAL([ENABLE_MODULE_ECDSA_S2C], [test x"$enable_module_ecdsa_s2c" = x"yes"])
AM_CONDITIONAL([ENABLE_MODULE_ECDSA_ADAPTOR], [test x"$enable_module_ecdsa_adaptor" = x"yes"])
AM_CONDITIONAL([USE_EXTERNAL_ASM], [test x"$enable_external_asm" = x"yes"])
AM_CONDITIONAL([USE_ASM_ARM], [test x"$set_asm" = x"arm"])
AM_CONDITIONAL([ENABLE_MODULE_SURJECTIONPROOF], [test x"$enable_module_surjectionproof" = x"yes"])
AM_CONDITIONAL([USE_REDUCED_SURJECTION_PROOF_SIZE], [test x"$use_reduced_surjection_proof_size" = x"yes"])
AM_CONDITIONAL([BUILD_WINDOWS], [test "$build_windows" = "yes"])
AC_SUBST(LIB_VERSION_CURRENT, _LIB_VERSION_CURRENT)
AC_SUBST(LIB_VERSION_REVISION, _LIB_VERSION_REVISION)
AC_SUBST(LIB_VERSION_AGE, _LIB_VERSION_AGE)

AC_OUTPUT

echo
echo "Build Options:"
echo "  with external callbacks = $enable_external_default_callbacks"
echo "  with benchmarks         = $enable_benchmark"
echo "  with tests              = $enable_tests"
echo "  with coverage           = $enable_coverage"
echo "  with examples           = $enable_examples"
echo "  module ecdh             = $enable_module_ecdh"
echo "  module recovery         = $enable_module_recovery"
echo "  module extrakeys        = $enable_module_extrakeys"
echo "  module schnorrsig       = $enable_module_schnorrsig"
echo "  module generator        = $enable_module_generator"
echo "  module rangeproof       = $enable_module_rangeproof"
echo "  module surjectionproof  = $enable_module_surjectionproof"
echo "  module whitelist        = $enable_module_whitelist"
echo "  module musig            = $enable_module_musig"
echo "  module ecdsa-s2c        = $enable_module_ecdsa_s2c"
echo "  module ecdsa-adaptor    = $enable_module_ecdsa_adaptor"
echo
echo "  asm                     = $set_asm"
echo "  ecmult window size      = $set_ecmult_window"
echo "  ecmult gen prec. bits   = $set_ecmult_gen_precision"
# Hide test-only options unless they're used.
if test x"$set_widemul" != xauto; then
echo "  wide multiplication     = $set_widemul"
fi
echo
echo "  valgrind                = $enable_valgrind"
echo "  CC                      = $CC"
echo "  CPPFLAGS                = $CPPFLAGS"
echo "  SECP_CFLAGS             = $SECP_CFLAGS"
echo "  CFLAGS                  = $CFLAGS"
echo "  LDFLAGS                 = $LDFLAGS"
