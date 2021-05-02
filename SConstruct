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
    cppflags += " /wd4101"

if staticlib:
    defs.append("LIBRAW_NODLL")
else:
    defs.append("LIBRAW_BUILDLIB")


lcms2_overrides = {}

# jpeg setup

def JpegLibname(static):
    return "jpeg"

rv = excons.ExternalLibRequire("libjpeg", libnameFunc=JpegLibname)
if not rv["require"]:
    if with_jpg:
        jpegStatic = (excons.GetArgument("libjpeg-static", 1, int) != 0)
        excons.PrintOnce("Build jpeg from sources ...")
        excons.Call("libjpeg-turbo", imp=["LibjpegName RequireLibjpeg"])
        def JpegRequire(env):
            RequireLibjpeg(env, static=jpegStatic) # pylint: disable=undefined-variable
        # If we are to build lcms2 from sources, have it use this jpeg library
        lcms2_overrides["with-libjpeg"] = excons.OutputBaseDirectory()
        lcms2_overrides["libjpeg-static"] = (1 if jpegStatic else 0)
        lcms2_overrides["libjpeg-name"] = LibjpegName(static=jpegStatic) # pylint: disable=undefined-variable
    else:
        def JpegRequire(env):
            pass
else:
    JpegRequire = rv["require"]

# LCMS2 setup

def Lcms2Defines(static):
    return (["CMS_DLL"] if not static else [])

rv = excons.ExternalLibRequire("lcms2", definesFunc=Lcms2Defines)
if not rv["require"]:
    if with_lcms2:
        lcms2Static = (excons.GetArgument("lcms2-static", 1, int) != 0)
        excons.PrintOnce("Build lcms2 from sources ...")
        excons.Call("Little-CMS", overrides=lcms2_overrides, imp=["RequireLCMS2"])
        def Lcms2Require(env):
            RequireLCMS2(env) # pylint: disable=undefined-variable
    else:
        def Lcms2Require(env):
            pass
else:
    Lcms2Require = rv["require"]




# libraw

if with_jpg:
    defs += ["USE_JPEG"]
    customs += [JpegRequire]

if with_lcms2:
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
    if staticlib:
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

prjs.append({"name": LibrawName(),
             "type": "staticlib" if staticlib else "sharedlib",
             "alias": "libraw",
             "defs": defs,
             "cppflags": cppflags,
             "incdirs": [".", "libraw", out_incdir],
             "srcs": ["internal/dcraw_common.cpp",
                      "internal/dcraw_fileio.cpp",
                      "internal/demosaic_packs.cpp",
                      "src/libraw_cxx.cpp",
                      "src/libraw_c_api.cpp",
                      "src/libraw_datastream.cpp"],
             "symvis": "default",
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

    base = os.path.splitext(os.path.basename(sam))[0]
    tests.append(base)
    prjs.append({"name": base,
                 "alias": "libraw-tests",
                 "type": "program",
                 "cppflags": cppflags,
                 "incdirs": [out_incdir],
                 "defs": defs,
                 "srcs": [sam],
                 "custom": [RequireLibraw]})

excons.AddHelpOptions(libraw="""LIBRAW OPTIONS
  libraw-static=0|1     : Toggle between static and shared library build [1]
  libraw-suffix=<str>   : Library name suffix. []
  libraw-with-jpeg=0|1  : Build with JPEG support. [0]
  libraw-with-lcms2=0|1 : Build with LCMS2 support. [0]""")
excons.AddHelpTargets({"libraw-tests": ", ".join(tests)})

excons.DeclareTargets(env, prjs)

SCons.Script.Export("LibrawName LibrawPath RequireLibraw")
