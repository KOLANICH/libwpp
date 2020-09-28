cmake_minimum_required(VERSION 3.18.0)
project(libwpp)

#include(CheckIncludeFiles)
include(CheckSymbolExists)
include(CheckStructHasMember)
include(CPackComponent)
include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

set(source_dir "${CMAKE_CURRENT_SOURCE_DIR}/src")
set(headers_dir "${CMAKE_CURRENT_SOURCE_DIR}/include")

find_package(BISON REQUIRED)
find_package(FLEX REQUIRED)

find_package(WinePort REQUIRED)

set(lexerSrc "${CMAKE_CURRENT_BINARY_DIR}/ppl.yy.c")
FLEX_TARGET(wpp_lexer "${source_dir}/ppl.l" "${lexerSrc}")

set(parserSrc "${CMAKE_CURRENT_BINARY_DIR}/ppy.tab.c")
BISON_TARGET(wpp_parser "${source_dir}/ppy.y" "${parserSrc}" COMPILE_FLAGS "--name-prefix=ppy_")

ADD_FLEX_BISON_DEPENDENCY(wpp_lexer wpp_parser)

set(target_name wpp)

file(GLOB_RECURSE SRCFILES "${source_dir}/*.c" "${source_dir}/*.cpp")

add_library("${target_name}" SHARED ${SRCFILES} ${BISON_wpp_parser_OUTPUTS} ${FLEX_wpp_lexer_OUTPUTS})

target_include_directories("${target_name}" PUBLIC "$<BUILD_INTERFACE:${headers_dir}>" "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"  "${WinePort_INCLUDE_DIRS}" PRIVATE "${source_dir}")
target_link_libraries("${target_name}" PRIVATE "${WinePort_LINK_LIBRARIES}")
set(LIB_FILE_NAME "${target_name}")

if(MSVC)
	if(MSVC_VERSION LESS 1900)
		target_add_definitions(
			-Dsnprintf=_snprintf
			-Dstrtoull=_strtoui64
			-Dstrtoll=_strtoi64)

		# Add this definition for WDK only, VS 9 doesn't like that
		if(DEFINED ENV{DDKBUILDENV})
			target_add_definitions("${target_name}" PRIVATE -Dvsnprintf=_vsnprintf)
		endif()
	endif()

	# Disable warning " unary minus operator applied to unsigned type, result still unsigned"
	add_compile_flags("/wd4146")

	# Disable warning "'=': conversion from 'a' to 'b', possible loss of data"
	add_compile_flags("/wd4244")

endif()

if(REACTOS AND CMAKE_CROSSCOMPILING)
	target_add_definitions("${target_name}" PRIVATE
		-D_DLL -D__USE_CRTIMP
		-D__NO_ISOCEXT
		-Dstrtoull=_strtoui64
		-Dstrtoll=_strtoi64
		-Dopen=_open
		-Dclose=_close)
	target_include_directories("${target_name}" "${REACTOS_SOURCE_DIR}/sdk/include/reactos/wine")
endif()


set(CPACK_PACKAGE_NAME "libwpp")
set(CPACK_PACKAGE_VENDOR "Wine, MinGW-w64 and ReactOS developers")
set(CPACK_PACKAGE_VERSION_MAJOR "5")
set(CPACK_PACKAGE_VERSION_MINOR "18")
set(CPACK_PACKAGE_VERSION_PATCH "0")
set(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")

set_target_properties(wpp PROPERTIES
	SOVERSION "${CPACK_PACKAGE_VERSION}"
)

set(CPACK_PACKAGE_DESCRIPTION "Wine preprocessor library")
set(CPACK_PACKAGE_HOMEPAGE_URL "https://github.com/KOLANICH/libwpp")
set(CPACK_PACKAGE_MAINTAINER "${CPACK_PACKAGE_VENDOR}")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "${CPACK_PACKAGE_MAINTAINER}")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.md")
set(CPACK_RESOURCE_FILE_README "${CMAKE_CURRENT_SOURCE_DIR}/ReadMe.md")
set(CPACK_PACKAGE_MAINTAINER "${CPACK_PACKAGE_VENDOR}")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "${CPACK_PACKAGE_MAINTAINER}")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.md")
set(CPACK_RESOURCE_FILE_README "${CMAKE_CURRENT_SOURCE_DIR}/ReadMe.md")
set(CPACK_DEB_COMPONENT_INSTALL ON)
set(CPACK_RPM_COMPONENT_INSTALL ON)
set(CPACK_NSIS_COMPONENT_INSTALL ON)
set("PKG_CONFIG_NAME" "libwpp")
set(CPACK_DEBIAN_PACKAGE_NAME "${PKG_CONFIG_NAME}")
set(CPACK_RPM_PACKAGE_NAME "${PKG_CONFIG_NAME}")
set(CPACK_DEBIAN_ENABLE_COMPONENT_DEPENDS ON)
set(CPACK_RPM_ENABLE_COMPONENT_DEPENDS ON)
set(CPACK_DEBIAN_COMPRESSION_TYPE "xz")
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
set(CPACK_DEBIAN_PACKAGE_GENERATE_SHLIBS ON)
set(CPACK_DEBIAN_PACKAGE_GENERATE_SHLIBS_POLICY ">=")

cpack_add_component(libwpp
	DISPLAY_NAME "library"
	DESCRIPTION "The library itself"
	REQUIRED
)

cpack_add_component(libwpp_dev
	DISPLAY_NAME "Development files"
	DESCRIPTION "Headers and other files needed for using the lib in own software"
	DEPENDS "libwpp"
)

set(CMAKE_EXPORT_NAME "libwpp")

# strangely, not populated automatically correctly
set(CPACK_COMPONENTS_ALL "libwpp;libwpp_dev")

install(TARGETS "${target_name}"
	LIBRARY
	COMPONENT "libwpp"
)
set("CPACK_DEBIAN_LIBWPP_PACKAGE_NAME" "${CPACK_DEBIAN_PACKAGE_NAME}")
set("CPACK_RPM_LIBWPP_PACKAGE_NAME" "${CPACK_RPM_PACKAGE_NAME}")

install(DIRECTORY "${headers_dir}/wine"
	TYPE INCLUDE
	COMPONENT "libwpp_dev"
)

set(PKG_CONFIG_FILE_NAME "${CMAKE_CURRENT_BINARY_DIR}/${PKG_CONFIG_NAME}.pc")
configure_file("${source_dir}/pkgconfig.pc.in" "${PKG_CONFIG_FILE_NAME}" @ONLY)
install(FILES "${PKG_CONFIG_FILE_NAME}"
	DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
	COMPONENT "libwpp_dev"
)

set(CMAKE_CONFIG_FILE_BASENAME "${CMAKE_EXPORT_NAME}Config.cmake")
set(CMAKE_CONFIG_VERSION_FILE_BASENAME "${CMAKE_EXPORT_NAME}ConfigVersion.cmake")
set(CMAKE_CONFIG_FILE_NAME "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_CONFIG_FILE_BASENAME}")
configure_package_config_file("${source_dir}/CMakeConfig.cmake.in" "${CMAKE_CONFIG_FILE_NAME}"
	INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake"
)


set(CMAKE_CONFIG_VERSION_FILE_NAME "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_CONFIG_VERSION_FILE_BASENAME}")
write_basic_package_version_file(
	"${CMAKE_CONFIG_VERSION_FILE_NAME}"
	VERSION "100500.100500.100500"  # any version of same bitness suits. CMake cannot compare to infinity, so use a large number we expect to be greater than any future version
	COMPATIBILITY AnyNewerVersion
)

install(FILES "${CMAKE_CONFIG_FILE_NAME}" "${CMAKE_CONFIG_VERSION_FILE_NAME}"
	#DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/wine"
	DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${CMAKE_EXPORT_NAME}"
	COMPONENT "libwpp_dev"
)

set("CPACK_DEBIAN_LIBWPP_DEV_PACKAGE_NAME" "${CPACK_DEBIAN_LIBWPP_PACKAGE_NAME}-dev")
set("CPACK_RPM_LIBWPP_DEV_PACKAGE_NAME" "${CPACK_RPM_LIBWPP_PACKAGE_NAME}-devel")

include(CPack)
