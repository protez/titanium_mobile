#!/usr/bin/env python
#
# Top level scons script
#
import os, shutil, platform, os.path as path, sys
import package
import SCons.Variables
import SCons.Environment
import subprocess
from SCons.Script import *

# read version from the build folder
# this is used by other python scripts too
cwd = os.path.abspath(os.path.dirname(sys._getframe(0).f_code.co_filename))
sys.path.append(path.join(cwd,"build"))
sys.path.append(path.join(cwd,"support","android"))
import titanium_version, ant
from androidsdk import AndroidSDK
version = titanium_version.version

# allow it to be overriden on command line or in env
if os.environ.has_key('PRODUCT_VERSION'):
	version = os.environ['PRODUCT_VERSION']
elif ARGUMENTS.get('PRODUCT_VERSION', 0):
	version = ARGUMENTS.get('PRODUCT_VERSION')

# get the githash for the build so we can always pull this build from a specific
# commit.  We're getting it here so we can pass it to android's ant build
# in order to get it into build.properties
p = subprocess.Popen(["git", "rev-parse", "HEAD"], stderr=subprocess.PIPE, stdout=subprocess.PIPE)
out, err = p.communicate()
if err:
	print >>sys.stderr, err
	print >>sys.stderr, "Error executing git rev-parse, is git in your PATH?"
	sys.exit(1)

githash = out.strip()[:7]
print "Building MobileSDK version %s, githash %s" % (version, githash)

#
# this is messy, but i don't care, scons makes it too
# hard to include python after an external SConscript
#

build_dirs = []

if not ARGUMENTS.get('iphone',0):
	build_dirs.append('android/titanium')

if platform.system() == "Darwin" and not ARGUMENTS.get('android',0):
	build_dirs.append('iphone')

flags = ''

only_package = False
if ARGUMENTS.get("package",0):
	only_package = True

install = "install" in COMMAND_LINE_TARGETS or ARGUMENTS.get("install", 0)
clean = "clean" in COMMAND_LINE_TARGETS or ARGUMENTS.get("clean", 0)
run_drillbit = "drillbit" in COMMAND_LINE_TARGETS or ARGUMENTS.get("drillbit",0)

if clean and os.path.exists('iphone/iphone/build'):
	shutil.rmtree('iphone/iphone/build')

# TEMP until android is merged
build_type = 'full'
build_dirs = ['iphone', 'android']
force_iphone = False

if ARGUMENTS.get('iphone',0):
	build_type='iphone'
	build_dirs=['iphone']

if ARGUMENTS.get('android',0):
	build_type='android'
	build_dirs=['android']

if ARGUMENTS.get('ipad',0):
	build_type='ipad'
	build_dirs=['ipad']

if ARGUMENTS.get('force_iphone',0):
	force_iphone = True

if ARGUMENTS.get('COMPILER_FLAGS', 0):
	flags = ARGUMENTS.get('COMPILER_FLAGS')

env = Environment()
Export("env cwd version")
if build_type in ['full', 'android'] and not only_package:
	d = os.getcwd()
	os.chdir('android')
	try:
		sdk = AndroidSDK(ARGUMENTS.get("android_sdk", None), 7)
		targets = ["full.build", "build.titanium.javadoc"]
		if clean: targets = ["clean"]
		elif "ant_targets" in ARGUMENTS: targets = ARGUMENTS["ant_targets"].split(",")
			
		ant.build(targets=targets, properties={"build.version": version, "build.githash": githash,
			"android.sdk": sdk.get_android_sdk(), "android.platform": sdk.get_platform_dir(), "google.apis": sdk.get_google_apis_dir()})
	finally:
		os.chdir(d)

if build_type in ['full', 'iphone', 'ipad'] and not only_package \
	and (platform.system() == "Darwin" or force_iphone):
	d = os.getcwd()
	os.chdir('iphone')
	try:
		#output = 0
		if clean: build_type = "clean"
		output = os.system("scons PRODUCT_VERSION=%s COMPILER_FLAGS='%s' BUILD_TYPE='%s'" % (version,flags,build_type))	
		if output!=0:
			sys.stderr.write("BUILD FAILED!!!!\n")
			# beep, please
			if platform.system() == "Darwin":
				os.system("say 'OH NO...the build failed!!!'")
				os.system("printf \"\a\"")
				os.system("printf \"\a\"")
				os.system("printf \"\a\"")
			sys.exit(output)
	finally:
		os.chdir(d)

def install_mobilesdk(version_tag):
	if (platform.system() == "Darwin"):
		os_names = { "Windows":"win32", "Linux":"linux", "Darwin":"osx" }
		os_name = os_names[platform.system()]
		mobilesdk_zipfile = os.path.join(os.path.abspath('dist'), 'mobilesdk-%s-%s.zip' % (version_tag, os_name))
		print "Installing %s..." % mobilesdk_zipfile
		installation_directory = os.path.expanduser('~/Library/Application Support/Titanium')
		os.system('/usr/bin/unzip -q -o -d "%s" "%s"' % (installation_directory, mobilesdk_zipfile))

def package_sdk(target, source, env):
	android = build_type in ['full', 'android']
	iphone = build_type in ['full', 'iphone']
	ipad = build_type in ['full', 'ipad']
	package_all = ARGUMENTS.get('package_all', 0)
	version_tag = ARGUMENTS.get('version_tag', version)
	print "Packaging MobileSDK (%s)..." % version_tag
	if package_all:
		package.Packager().build_all_platforms(os.path.abspath('dist'), version, android, iphone, ipad, version_tag)
	else:
		package.Packager().build(os.path.abspath('dist'), version, android, iphone, ipad, version_tag)
	if install and not clean:
		install_mobilesdk(version_tag)

def drillbit_builder(target, source, env):
	sys.path.append("drillbit")
	import drillbit
	drillbit.build_and_run(android_sdk=sdk.get_android_sdk())

if run_drillbit:
	drillbit = Builder(action = drillbit_builder)
	env.Append(BUILDERS = {'BuildAndRunDrillbit': drillbit})
	env.BuildAndRunDrillbit('#dummy-drillbit-target', [])

package_builder = Builder(action = package_sdk)
env.Append(BUILDERS = {'PackageMobileSDK': package_builder})
env.PackageMobileSDK("#dummy-sdk-target", [])
	
if clean:
	# don't error 
	Exit(0)
