import os
import re
import sys
import glob
import tarfile
import excons
import SCons.Script # pylint: disable=import-error


env = excons.MakeBaseEnv()


staticlib = (excons.GetArgument("libraw-static", 1, int) != 0)
libsuffix = excons.GetArgument("libraw-suffix", "")
with_jpg = (excons.GetArgument("libraw-with-jpeg", 0, int) != 0)
with_lcms2 = (excons.GetArgument("libraw-with-lcms2", 0, int) != 0)

out_incdir = excons.OutputBaseDirectory() + "/include"
out_libdir = excons.OutputBaseDirectory() + "/lib"

#excons.SetArgument("libjpeg-jpeg8", 1)

prjs = []
defs = []
customs = []
cppflags = ""

major = minor = patch = so_cur = so_rev = so_age = None
with open("libraw/libraw_version.h", "r") as f:
    for l in f.readlines():
        res = re.search(r"#define\sLIBRAW_MAJOR_VERSION\s+(\d+)+", l)
        if res:
            major = res.group(1)

        res = re.search(r"#define\sLIBRAW_MINOR_VERSION\s+(\d+)+", l)
        if res:
            minor = res.group(1)

        res = re.search(r"#define\sLIBRAW_PATCH_VERSION\s+(\d+)+", l)
        if res:
            patch = res.group(1)

        res = re.search(r"#define\sLIBRAW_SHLIB_CURRENT\s+(\d+)+", l)
        if res:
            so_cur = res.group(1)

        res = re.search(r"#define\sLIBRAW_SHLIB_REVISION\s+(\d+)+", l)
        if res:
            so_rev = res.group(1)

        res = re.search(r"#define\sLIBRAW_SHLIB_AGE\s+(\d+)+", l)
        if res:
            so_age = res.group(1)

if major is None or minor is None or patch is None or so_cur is None or so_rev is None or so_age is None:
    print ".. parsing libraw/libraw_version.h failed."
    sys.exit(1)

if sys.platform != "win32":
    cppflags += " -Wno-unused-parameter"
    cppflags += " -Wno-unused-variable"
    cppflags += " -Wno-missing-field-initializers"
    cppflags += " -Wno-unused-function"
    cppflags += " -Wno-sign-compare"

    if sys.platform != "darwin":
        cppflags += " -Wno-unused-but-set-variable"
        cppflags += " -Wno-parentheses"
        cppflags += " -Wno-type-limits"
        cppflags += " -Wno-narrowing"
        cppflags += " -Wno-maybe-uninitialized"
        cppflags += " -Wno-aggressive-loop-optimizations"
    else:
        cppflags += " -Wno-unused-private-field"
        cppflags += " -Wno-constant-conversion"
        cppflags += " -Wno-sometimes-uninitialized"
        cppflags += " -Wno-deprecated-declarations"

else:
    # 4101: ?
    # 4706: assignment within conditional expression
    # 4702: unreachable code
    # 4244: conversion from X to Y, possible loss of data
    # 4267: conversion from X to Y, possible loss of data
    # 4389: signed/unsigned mismatch (assignment)
    # 4018: signed/unsigned mismatch (comparison)
    # 4305: truncation from 'double' to 'float'
    # 4127: conditional expression is constant
    # 4819: The file contains a character that cannot be represented in the current code page
    # 4146: unary minus operator applied to unsigned type, result still unsigned
    # 4804: unsafe use of type 'bool' in operation
    # 4100: unreferenced formal parameter
    # 4005: macro redefinition
    # 4456: declaration of X hides previous local declaration
    # 4838: conversion from X to Y requires a narrowing conversion
    # 4701: potentially uninitialized local variable X used
    # 4245: conversion from X to Y, signed/unsigned mismatch
    # 4458: declaration of X hides class member
    # 4324: structure was padded due to alignment specifier
    # 4703: potentially uninitialized local pointer variable 'LensData_buf' used
    # 4457: declaration of X hides function parameter
    # 4611: interaction between '_setjmp' and C++ object destruction is non-portable
    cppflags += " /wd4324 /wd4703 /wd4457 /wd4611 /wd4458 /wd4245 /wd4101 /wd4706 /wd4702 /wd4244 /wd4389 /wd4305 /wd4127 /wd4819 /wd4018 /wd4146 /wd4804 /wd4100 /wd4005 /wd4456 /wd4267 /wd4838 /wd4701"

if sys.platform == "win32":
    if staticlib:
        defs.append("LIBRAW_NODLL")
    else:
        defs.append("LIBRAW_BUILDLIB")

lcms2_overrides = {}
tiff_deps = []

if with_jpg:
    # jpeg setup
    def JpegLibname(static):
        return "jpeg"

    rv = excons.ExternalLibRequire("libjpeg", libnameFunc=JpegLibname)
    if not rv["require"]:
        jpegStatic = (excons.GetArgument("libjpeg-static", 1, int) != 0)
        excons.PrintOnce("Build jpeg from sources ...")
        excons.Call("libjpeg-turbo", targets=["libjpeg"], imp=["LibjpegName", "LibjpegPath", "RequireLibjpeg"])
        def JpegRequire(env):
            RequireLibjpeg(env, static=jpegStatic) # pylint: disable=undefined-variable
        # If we are to build lcms2 from sources, have it use this jpeg library
        lcms2_overrides["with-libjpeg"] = excons.OutputBaseDirectory()
        lcms2_overrides["libjpeg-static"] = (1 if jpegStatic else 0)
        lcms2_overrides["libjpeg-name"] = LibjpegName(static=jpegStatic) # pylint: disable=undefined-variable
        # lcms2 may build libtiff which also requires libjpeg, add libjpeg.cmake.outputs/libjpeg.automake.outputs
        #   as a configure dependency to ensure libjpeg is fully built before libtiff is
        if sys.platform == "win32":
            tiff_deps.append(excons.cmake.OutputsCachePath("libjpeg"))
        else:
            tiff_deps.append(excons.automake.OutputsCachePath("libjpeg"))
    else:
        JpegRequire = rv["require"]

    defs += ["USE_JPEG"]
    customs += [JpegRequire]

if with_lcms2:
    # LCMS2 setup
    def Lcms2Defines(static):
        return (["CMS_DLL"] if not static else [])

    rv = excons.ExternalLibRequire("lcms2", definesFunc=Lcms2Defines)
    if not rv["require"]:
        lcms2Static = (excons.GetArgument("lcms2-static", 1, int) != 0)
        excons.PrintOnce("Build lcms2 from sources ...")
        if with_jpg:
            excons.cmake.AddConfigureDependencies("libtiff", tiff_deps)
        excons.Call("Little-CMS", targets=["lcms2"], overrides=lcms2_overrides, imp=["RequireLCMS2", "LCMS2Path"])
        def Lcms2Require(env):
            RequireLCMS2(env) # pylint: disable=undefined-variable
    else:
        Lcms2Require = rv["require"]

    defs += ["USE_LCMS2"]
    customs += [Lcms2Require]


def LibrawName():
    name = "raw" + libsuffix
    if sys.platform == "win32" and staticlib:
        name = "lib" + name
    return name

def LibrawPath():
    name = LibrawName()
    if sys.platform == "win32":
        libname = name + ".lib"
    else:
        libname = "lib" + name + (".a" if staticlib else excons.SharedLibraryLinkExt())
    return out_libdir + "/" + libname

def RequireLibraw(env):
    if staticlib and sys.platform == "win32":
        env.Append(CPPDEFINES=["LIBRAW_NODLL"])
    if with_lcms2:
        env.Append(CPPDEFINES=["USE_LCMS2"])
    env.Append(CPPPATH=[out_incdir])
    env.Append(LIBPATH=[out_libdir])
    excons.Link(env, LibrawPath(), static=staticlib, force=True, silent=True)
    if staticlib:
        if with_jpg:
            JpegRequire(env)
        if with_lcms2:
            Lcms2Require(env)

def GlobNoPH(pat):
   def _flt(item):
      bn = os.path.splitext(os.path.basename(str(item)))[0]
      return (not bn.endswith("_ph"))
   return filter(_flt, SCons.Script.Glob(pat))

prjs.append({"name": LibrawName(),
             "type": "staticlib" if staticlib else "sharedlib",
             "alias": "libraw",
             "defs": defs,
             "cppflags": cppflags,
             "incdirs": [".", "libraw", out_incdir],
             "srcs": ["src/libraw_c_api.cpp", "src/libraw_datastream.cpp"] +
                      GlobNoPH("src/decoders/*.cpp") +
                      GlobNoPH("src/demosaic/*.cpp") +
                      GlobNoPH("src/integration/*.cpp") +
                      GlobNoPH("src/metadata/*.cpp") +
                      GlobNoPH("src/postprocessing/*.cpp") +
                      GlobNoPH("src/preprocessing/*.cpp") +
                      GlobNoPH("src/tables/*.cpp") +
                      GlobNoPH("src/utils/*.cpp") +
                      GlobNoPH("src/write/*.cpp") +
                      GlobNoPH("src/x3f/*.cpp"),
             "version": "%s.%s.%s" % (major, minor, patch),
             "soname": "lib%s.so.%s" % (LibrawName(), major),
             "install_name": "lib%s.%s.dylib" % (LibrawName(), major),
             "install": {"%s/libraw" % out_incdir: excons.glob("libraw/*.h")},
             "custom": customs})

tests = []
for sam in excons.glob("samples/*.cpp"):
    if "unprocessed_raw.cpp" in sam:
        if sys.platform == "win32":
            continue

    customs = [RequireLibraw]
    if "mem_image_sample" in sam and not staticlib and with_jpg:
        # mem_image_sample uses libjpeg directly but linking libraw dynamically
        # -> require jpeg library explicitely
        customs.append(JpegRequire)

    base = os.path.splitext(os.path.basename(sam))[0]
    tests.append(base)
    prjs.append({"name": base,
                 "alias": "libraw-tests",
                 "type": "program",
                 "cppflags": cppflags,
                 "incdirs": [out_incdir],
                 "defs": defs,
                 "srcs": [sam],
                 "custom": customs})

excons.AddHelpOptions(libraw="""LIBRAW OPTIONS
  libraw-static=0|1     : Toggle between static and shared library build [1]
  libraw-suffix=<str>   : Library name suffix. []
  libraw-with-jpeg=0|1  : Build with JPEG support. [0]
  libraw-with-lcms2=0|1 : Build with LCMS2 support. [0]""")
excons.AddHelpTargets({"libraw-tests": ", ".join(tests)})

excons.DeclareTargets(env, prjs)

SCons.Script.Export("LibrawName LibrawPath RequireLibraw")
