#EQEmu Cmake

#We set a fairly new version (as of 2013) because I found finding perl was a bit... buggy on older ones
#Can change this if you really want but you should upgrade!
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

#FindMySQL is located here so lets make it so CMake can find it
SET(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/" ${CMAKE_MODULE_PATH})

#For checking includes
INCLUDE (CheckIncludeFiles)

#Our project name is EQEmu
PROJECT(EQEmu)

#Default build type is set to RelWithDebInfo for generators that honor that like makefiles
IF(NOT CMAKE_BUILD_TYPE)
	SET(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Choose the type of build." FORCE)
ENDIF(NOT CMAKE_BUILD_TYPE)

#Add our various windows definitions
IF(MSVC OR MINGW)
	ADD_DEFINITIONS(-D_WINDOWS)
	IF(CMAKE_CL_64)
		ADD_DEFINITIONS(-DWIN64)
	ELSE(CMAKE_CL_64)
		ADD_DEFINITIONS(-DWIN32)
	ENDIF(CMAKE_CL_64)
ENDIF(MSVC OR MINGW)

IF(MSVC)
	#Set our default locations for zlib/mysql based on x86/x64
	IF(CMAKE_CL_64)
		SET(ZLIB_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/zlib_x64")
		SET(MYSQL_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/mysql_x64")
		SET(LUA_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/luaj_x64")
	ELSE(CMAKE_CL_64)
		SET(ZLIB_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/zlib_x86")
		SET(MYSQL_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/mysql_x86")
		SET(LUA_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/luaj_x86")
	ENDIF(CMAKE_CL_64)

	#disable CRT warnings on windows cause they're annoying as shit and we use C functions everywhere
	OPTION(EQEMU_DISABLE_CRT_SECURE_WARNINGS "Disable Secure CRT Warnings" ON)
	IF(EQEMU_DISABLE_CRT_SECURE_WARNINGS)
		ADD_DEFINITIONS(-D_CRT_SECURE_NO_WARNINGS)
	ENDIF(EQEMU_DISABLE_CRT_SECURE_WARNINGS)

	#fast FP if you'd like it
	OPTION(EQEMU_FAST_FLOATINGPOINT "Use MSVC /fp:fast option" ON)
	IF(EQEMU_FAST_FLOATINGPOINT)
		ADD_DEFINITIONS(/fp:fast)
	ENDIF(EQEMU_FAST_FLOATINGPOINT)

	#crash logging currently only works on windows x86/x64
	OPTION(EQEMU_ENABLE_CRASH_LOGGING "Enable crash logging" ON)
	IF(EQEMU_ENABLE_CRASH_LOGGING)
		ADD_DEFINITIONS(-DCRASH_LOGGING)
	ENDIF(EQEMU_ENABLE_CRASH_LOGGING)

	#Disable safe SEH or not?
	OPTION(EQEMU_DISABLE_SAFESEH "Disable Safe SEH (Needed for Strawberry Perl)" OFF)
	IF(EQEMU_DISABLE_SAFESEH)
		SET(CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG} /SAFESEH:NO")
		SET(CMAKE_EXE_LINKER_FLAGS_MINSIZEREL "${CMAKE_EXE_LINKER_FLAGS_MINSIZEREL} /SAFESEH:NO")
		SET(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /SAFESEH:NO")
		SET(CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO "${CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO} /SAFESEH:NO")
		SET(CMAKE_SHARED_LINKER_FLAGS_DEBUG "${CMAKE_SHARED_LINKER_FLAGS_DEBUG} /SAFESEH:NO")
		SET(CMAKE_SHARED_LINKER_FLAGS_MINSIZEREL "${CMAKE_SHARED_LINKER_FLAGS_MINSIZEREL} /SAFESEH:NO")
		SET(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} /SAFESEH:NO")
		SET(CMAKE_SHARED_LINKER_FLAGS_RELWITHDEBINFO "${CMAKE_SHARED_LINKER_FLAGS_RELWITHDEBINFO} /SAFESEH:NO")
		SET(CMAKE_MODULE_LINKER_FLAGS_DEBUG "${CMAKE_MODULE_LINKER_FLAGS_DEBUG} /SAFESEH:NO")
		SET(CMAKE_MODULE_LINKER_FLAGS_MINSIZEREL "${CMAKE_MODULE_LINKER_FLAGS_MINSIZEREL} /SAFESEH:NO")
		SET(CMAKE_MODULE_LINKER_FLAGS_RELEASE "${CMAKE_MODULE_LINKER_FLAGS_RELEASE} /SAFESEH:NO")
		SET(CMAKE_MODULE_LINKER_FLAGS_RELWITHDEBINFO "${CMAKE_MODULE_LINKER_FLAGS_RELWITHDEBINFO} /SAFESEH:NO")
	ENDIF(EQEMU_DISABLE_SAFESEH)

	#We want to compile /MT not /MD so we change that
	FOREACH(flag_var CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELEASE CMAKE_CXX_FLAGS_RELWITHDEBINFO CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELEASE CMAKE_C_FLAGS_RELWITHDEBINFO)
		IF(${flag_var} MATCHES "/MD")
			STRING(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
		ENDIF(${flag_var} MATCHES "/MD")
	ENDFOREACH(flag_var)
	
	ADD_DEFINITIONS(-DNOMINMAX)
ELSE(MSVC)
	#Normally set by perl but we don't use the perl flags anymore so we set it.
	ADD_DEFINITIONS(-DHAS_UNION_SEMUN)
ENDIF(MSVC)

#FreeBSD support
IF(UNIX)
	IF(CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
		ADD_DEFINITIONS(-DFREEBSD)
		SET(FREEBSD TRUE)
	ENDIF(CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
ENDIF(UNIX)

#use stdint.h types if they exist for this platform (we have to guess otherwise)
CHECK_INCLUDE_FILES(stdint.h HAVE_STDINT_H)
IF(HAVE_STDINT_H)
	ADD_DEFINITIONS(-DEQEMU_USE_STDINT)
ENDIF(HAVE_STDINT_H)

#debug level, 5 is default. Most people wont ever change this but it's there if you want to
SET(EQEMU_DEBUG_LEVEL 5 CACHE STRING "EQEmu debug level:
	0 - Quiet mode Errors to file Status and Normal ignored
	1 - Status and Normal to console, Errors to logfile
	2 - Status, Normal, and Error to console and logfile
	3 - Light debug release errors and status
	4 - Moderate debug release errors and status
	5 - Maximum debug release errors and status
	10 - More errors than you ever wanted to see"
)

#Bots are a compile time option so on/off
OPTION(EQEMU_ENABLE_BOTS "Enable Bots" OFF)

#Disable entire _mlog system (excludes trade/command logs)
OPTION(EQEMU_DISABLE_LOGSYS "Disable Logging INI System" ON)

#Enable GM Command log system
OPTION(EQEMU_COMMANDS_LOGGING "Enable GM Command logs" ON)

IF(EQEMU_COMMANDS_LOGGING)
	ADD_DEFINITIONS(-DCOMMANDS_LOGGING)
ENDIF(EQEMU_COMMANDS_LOGGING)

IF(EQEMU_DISABLE_LOGSYS)
	ADD_DEFINITIONS(-DDISABLE_LOGSYS)
ENDIF(EQEMU_DISABLE_LOGSYS)

IF(EQEMU_ENABLE_BOTS)
	ADD_DEFINITIONS(-DBOTS)
ENDIF(EQEMU_ENABLE_BOTS)

#What to build
OPTION(EQEMU_BUILD_SERVER "Build the game server." ON)
OPTION(EQEMU_BUILD_LOGIN "Build the login server." OFF)
OPTION(EQEMU_BUILD_AZONE "Build azone utility." OFF)
OPTION(EQEMU_BUILD_TESTS "Build utility tests." OFF)
OPTION(EQEMU_BUILD_PERL "Build Perl parser." ON)
OPTION(EQEMU_BUILD_LUA "Build Lua parser." OFF)
OPTION(EQEMU_BUILD_CLIENT_FILES "Build Client Inport/Export Data Programs." ON)

#C++11 stuff
IF(NOT MSVC)
	ADD_DEFINITIONS(-std=c++0x)
ENDIF(NOT MSVC)

#Various definitions
IF(EQEMU_BUILD_PERL)
	ADD_DEFINITIONS(-DEMBPERL)
	ADD_DEFINITIONS(-DEMBPERL_PLUGIN)
ENDIF(EQEMU_BUILD_PERL)
IF(EQEMU_BUILD_LUA)
	ADD_DEFINITIONS(-DLUA_EQEMU)
ENDIF(EQEMU_BUILD_LUA)

ADD_DEFINITIONS(-DEQDEBUG=${EQEMU_DEBUG_LEVEL})
ADD_DEFINITIONS(-DINVERSEXY)
ADD_DEFINITIONS(-DFIELD_ITEMS)
ADD_DEFINITIONS(-DMAP_DIR="./Maps")

#Find everything we need
FIND_PACKAGE(ZLIB REQUIRED)
FIND_PACKAGE(MySQL REQUIRED)
IF(EQEMU_BUILD_PERL)
	FIND_PACKAGE(PerlLibs REQUIRED)
	INCLUDE_DIRECTORIES("${PERL_INCLUDE_PATH}")
ENDIF(EQEMU_BUILD_PERL)

IF(EQEMU_BUILD_LUA)
	FIND_PACKAGE(EQLua51 REQUIRED)
	SET(Boost_USE_STATIC_LIBS OFF)
	SET(Boost_USE_MULTITHREADED ON) 
	SET(Boost_USE_STATIC_RUNTIME OFF)
	SET(BOOST_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/boost")

	FIND_PACKAGE(Boost REQUIRED)
	INCLUDE_DIRECTORIES("${LUA_INCLUDE_DIR}" "${Boost_INCLUDE_DIRS}" "luabind")
	
	OPTION(EQEMU_SANITIZE_LUA_LIBS "Sanitize Lua Libraries (Remove OS and IO standard libraries from being able to run)." ON)
	IF(EQEMU_SANITIZE_LUA_LIBS)
		ADD_DEFINITIONS(-DSANITIZE_LUA_LIBS)
	ENDIF(EQEMU_SANITIZE_LUA_LIBS)
ENDIF(EQEMU_BUILD_LUA)

INCLUDE_DIRECTORIES("${ZLIB_INCLUDE_DIRS}" "${MySQL_INCLUDE_DIR}")

IF(EQEMU_BUILD_LUA)
    ADD_SUBDIRECTORY(luabind)
ENDIF(EQEMU_BUILD_LUA)

IF(EQEMU_BUILD_SERVER OR EQEMU_BUILD_LOGIN OR EQEMU_BUILD_TESTS)
	ADD_SUBDIRECTORY(common)
ENDIF(EQEMU_BUILD_SERVER OR EQEMU_BUILD_LOGIN OR EQEMU_BUILD_TESTS)
IF(EQEMU_BUILD_SERVER)
	ADD_SUBDIRECTORY(shared_memory)
	ADD_SUBDIRECTORY(world)
	ADD_SUBDIRECTORY(zone)
	ADD_SUBDIRECTORY(ucs)
	ADD_SUBDIRECTORY(queryserv)
	ADD_SUBDIRECTORY(eqlaunch)
ENDIF(EQEMU_BUILD_SERVER)
IF(EQEMU_BUILD_LOGIN)
	ADD_SUBDIRECTORY(loginserver)
ENDIF(EQEMU_BUILD_LOGIN)

IF(EQEMU_BUILD_AZONE)
	ADD_SUBDIRECTORY(utils)
ENDIF(EQEMU_BUILD_AZONE)

IF(EQEMU_BUILD_TESTS)
	ADD_SUBDIRECTORY(tests)
ENDIF(EQEMU_BUILD_TESTS)

IF(EQEMU_BUILD_CLIENT_FILES)
	ADD_SUBDIRECTORY(client_files)
ENDIF(EQEMU_BUILD_CLIENT_FILES)
