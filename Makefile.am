SUBDIRS = src

EXTRA_DIST = autogen.sh README.rst debian

# Clear dependency_libs for multiarch
all-local:
	sed -i "/dependency_libs/ s/'.*'/''/" `find . -name '*.la'`
	sed -i "/dependency_libs/ s/'.*'/''/" `find . -name '*.lai'`

