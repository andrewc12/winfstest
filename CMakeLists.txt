cmake_minimum_required(VERSION 3.13)
project(Winfstest LANGUAGES C CXX ASM)
SET (CMAKE_ASM_COMPILER_ID Clang)
SET (CMAKE_ASM_COMPILER clang-cl.exe)

include(GNUInstallDirs)

# message(STATUS "OpenSSL libs: " ${OPENSSL_LIBRARIES})
# Seriously, why does it pick MD versions and not MT like requested?
# string(REGEX REPLACE "64MD" "64MT" INTERNAL_OPENSSL_LIBRARIES ${OPENSSL_LIBRARIES})
#message(STATUS "OpenSSL libs: " ${OPENSSL_CRYPTO_LIBRARY})
#cmake_path(GET ${OPENSSL_LIBRARIES} ROOT_PATH rootPath)
#get_filename_component(rootPath "${OPENSSL_CRYPTO_LIBRARY}" DIRECTORY)
#message(STATUS "OpenSSL root: " ${rootPath})




function(use_clang)
# It seems "-include file" is "/FI file" on Windows
set(CMAKE_C_COMPILER clang-cl.exe PARENT_SCOPE)
set(CMAKE_CXX_COMPILER clang-cl.exe PARENT_SCOPE)
if(CMAKE_SIZEOF_VOID_P EQUAL 4)
	add_compile_options(-m32)
else()
	add_compile_options(-m64)
endif()

add_compile_options(
	# These are all MS headers
	-Wno-nonportable-include-path
	-Wno-unknown-pragmas
	-Wno-ignored-pragma-intrinsic
	-Wno-pragma-pack
	-Wno-microsoft-enum-forward-reference
	-Wno-visibility
	-Wno-microsoft-anon-tag
	# -Wno-ignored-attributes
	-Wno-unused-command-line-argument
	-Wno-unused-local-typedef
	-Wno-int-to-void-pointer-cast
	# ZFS related
	-Wno-misleading-indentation
	-Wno-dangling-else
	-Wno-missing-field-initializers
	#-fms-extensions
	-Wno-unused-function
	-Wno-unused-label
	#Visual Studio 2022 update 2022114 started breaking builds with errors
	-Wno-int-conversion
)
endfunction()

if(CMAKE_SIZEOF_VOID_P EQUAL 4)
    add_definitions(-D__x86_32__ -D__i386 -D__i386__ -D_LP32 -DWIN32)
elseif(CMAKE_SIZEOF_VOID_P EQUAL 8)
    add_definitions(-D__x86_64__ -D_LP64 -D__LP64__ -D__x86_64 -D_AMD64_ -D_WIN64 -DAMD64 -DWIN64)
else()
    message(FATAL_ERROR "Unsupported architecture")
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_definitions(-DDBG -DZFS_DEBUG)
endif()

# Avoid dependency add_compile_definitions vcruntime140.dll
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /MT")
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /MTd")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /MT")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")

# Let's try to emulate gitrev
#find_path(BASH_DIR NAMES bash.exe git-bash.exe PATHS "/c/Program Files (x86)/Git/")
#execute_process(COMMAND $BASH_DIR -c "scripts/make_gitrev.h include/zfs_gitrev.h" )


function(um_add_executable name)
	add_executable(${ARGV})

	target_link_options(${name} PRIVATE "/MANIFESTUAC:level=\"requireAdministrator\" uiAccess=\"false\"")
	add_custom_command(
		TARGET ${name}
		POST_BUILD
		#COMMAND "${SIGNTOOL_PROGRAM}" sign /v /fd sha1 /s "${ZFSIN_SIGNTOOL_CERTSTORE}" /n "${ZFSIN_SIGNTOOL_CERTNAME}" /t "${ZFSIN_SIGNTOOL_TSA}" $<TARGET_FILE:${name}>
		#COMMAND "${SIGNTOOL_PROGRAM}" sign /v /as /fd sha256 /s "${ZFSIN_SIGNTOOL_CERTSTORE}" /n "${ZFSIN_SIGNTOOL_CERTNAME}" /tr "${ZFSIN_SIGNTOOL_TSA}" $<TARGET_FILE:${name}>
		#COMMAND "${SIGNTOOL_PROGRAM}" sign /v /fd sha1 /sha1 "${ZFSIN_SIGNTOOL_SHA1}" /t "${ZFSIN_SIGNTOOL_TSA}" $<TARGET_FILE:${name}>
		# Lundman's line
		COMMAND "${SIGNTOOL_PROGRAM}" sign /v /as /fd sha256 /td sha256 /sha1 "${ZFSIN_SIGNTOOL_SHA1}" /tr "${ZFSIN_SIGNTOOL_TSA}" $<TARGET_FILE:${name}>
		WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
		VERBATIM COMMENT "Sign userspace tool"
	)
endfunction()



use_clang()

um_add_executable(winfstest
	winfstest.c
)

target_link_libraries(winfstest PRIVATE
)
install(TARGETS winfstest RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
install(FILES $<TARGET_PDB_FILE:winfstest>
	DESTINATION "${CMAKE_INSTALL_BINDIR}"
	OPTIONAL
)
