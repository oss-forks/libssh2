
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

##################################################################################################################
## zlib

FIND_PACKAGE(ZLIB)
IF(ZLIB_FOUND)
	INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIRS})
	SET(LIBSSH2_HAVE_ZLIB TRUE)
ELSE()
	SET(LIBSSH2_HAVE_ZLIB FALSE)
ENDIF()

INCLUDE("${ZLIB_INCLUDE_DIRS}/zlib.cmake" OPTIONAL)

##################################################################################################################
## openssl

FIND_PACKAGE(OpenSSL REQUIRED)
IF(OPENSSL_FOUND)
	INCLUDE_DIRECTORIES(${OPENSSL_INCLUDE_DIR})
ENDIF()

INCLUDE("${OPENSSL_INCLUDE_DIR}/openssl.cmake" OPTIONAL)

##################################################################################################################
## compiler flags

INCLUDE(CheckTypeSize)
INCLUDE(CheckCSourceCompiles)
INCLUDE(CheckIncludeFile)
INCLUDE(CheckFunctionExists)

CHECK_TYPE_SIZE("long long" LONGLONG)

CHECK_INCLUDE_FILE(alloca.h HAVE_ALLOCA_H)
CHECK_INCLUDE_FILE(inttypes.h HAVE_INTTYPES_H)
CHECK_INCLUDE_FILE(malloc.h HAVE_MALLOC_H)
CHECK_INCLUDE_FILE(sys/ioctl.h HAVE_SYS_IOCTL_H)
CHECK_INCLUDE_FILE(sys/select.h HAVE_SYS_SELECT_H)
CHECK_INCLUDE_FILE(sys/socket.h HAVE_SYS_SOCKET_H)
CHECK_INCLUDE_FILE(sys/time.h HAVE_SYS_TIME_H)
CHECK_INCLUDE_FILE(sys/uio.h HAVE_SYS_UIO_H)
CHECK_INCLUDE_FILE(sys/un.h HAVE_SYS_UN_H)
CHECK_INCLUDE_FILE(unistd.h HAVE_UNISTD_H)
CHECK_INCLUDE_FILE(windows.h HAVE_WINDOWS_H)
CHECK_INCLUDE_FILE(winsock.h HAVE_WINSOCK_H)
CHECK_INCLUDE_FILE(winsock2.h HAVE_WINSOCK2_H)
CHECK_INCLUDE_FILE(ws2tcpip.h HAVE_WS2TCPIP_H)

CHECK_FUNCTION_EXISTS(gettimeofday HAVE_GETTIMEOFDAY)
CHECK_FUNCTION_EXISTS(poll HAVE_POLL)
CHECK_FUNCTION_EXISTS(select HAVE_SELECT)
CHECK_FUNCTION_EXISTS(strtoll HAVE_STRTOLL)

IF(HAVE_ALLOCA_H)
	LIST(APPEND CMAKE_REQUIRED_DEFINITIONS "-DHAVE_ALLOCA_H=1")
ENDIF()

IF(HAVE_MALLOC_H)
	LIST(APPEND CMAKE_REQUIRED_DEFINITIONS "-DHAVE_MALLOC_H=1")
ENDIF()

IF(HAVE_WINDOWS_H)
	LIST(APPEND CMAKE_REQUIRED_DEFINITIONS "-DHAVE_WINDOWS_H=1")
ENDIF()

IF(HAVE_WINSOCK_H)
	LIST(APPEND CMAKE_REQUIRED_DEFINITIONS "-DHAVE_WINSOCK_H=1")
ENDIF()

IF(HAVE_WINSOCK2_H)
	LIST(APPEND CMAKE_REQUIRED_DEFINITIONS "-DHAVE_WINSOCK2_H=1")
ENDIF()

CHECK_C_SOURCE_COMPILES("
#ifdef HAVE_ALLOCA_H
#include <alloca.h>
#endif
#ifdef HAVE_MALLOC_H
#include <malloc.h>
#endif
#include <stdlib.h>
int main(int argc, char ** argv)
{
	void * p = alloca(argc);
	return argc;
}"
HAVE_ALLOCA)

CHECK_C_SOURCE_COMPILES("
#include <sys/types.h>
#include <unistd.h>
#include <fcntl.h>

#if defined(sun) || defined(__sun__) || defined(__SUNPRO_C) || defined(__SUNPRO_CC)
# if defined(__SVR4) || defined(__srv4__)
#  define PLATFORM_SOLARIS
# else
#  define PLATFORM_SUNOS4
# endif
#endif
#if (defined(_AIX) || defined(__xlC__)) && !defined(_AIX41)
# define PLATFORM_AIX_V3
#endif

#if defined(PLATFORM_SUNOS4) || defined(PLATFORM_AIX_V3) || defined(__BEOS__)
#error \"O_NONBLOCK does not work on this platform\"
#endif

int main(int argc, char ** argv)
{
	int socket = argc;
	int flags = fcntl(socket, F_SETFL, flags | O_NONBLOCK);
	return flags;
}"
HAVE_O_NONBLOCK)

CHECK_C_SOURCE_COMPILES("
#include <unistd.h>
#include <stropts.h>
int main(int argc, char ** argv)
{
	int socket = argc;
	int flags = ioctl(socket, FIONBIO, &flags);
	return flags;
}"
HAVE_FIONBIO)

CHECK_C_SOURCE_COMPILES("
#undef inline
#ifdef HAVE_WINDOWS_H
#ifndef WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN
#endif
#include <windows.h>
#ifdef HAVE_WINSOCK2_H
#include <winsock2.h>
#else
#ifdef HAVE_WINSOCK_H
#include <winsock.h>
#endif
#endif
#endif
int main()
{
	SOCKET sd;
	unsigned long flags = 0;
	sd = socket(0, 0, 0);
	ioctlsocket(sd, FIONBIO, &flags);
	return 0;
}"
HAVE_IOCTLSOCKET)

CHECK_C_SOURCE_COMPILES("
#include <sys/ioctl.h>
int main(int argc, char ** argv)
{
	/* IoctlSocket source code */
	int socket = argc;
	int flags = IoctlSocket(socket, FIONBIO, (long)1);
	return flags;
}"
HAVE_IOCTLSOCKET_CASE)

CHECK_C_SOURCE_COMPILES("
#include <sys/socket.h>
int main(int argc, char ** argv)
{
	long b = 1;
	int socket = argc;
	int flags = setsockopt(socket, SOL_SOCKET, SO_NONBLOCK, &b, sizeof(b));
	return flags;
}"
HAVE_SO_NONBLOCK)

IF(NOT HAVE_O_NONBLOCK AND
		NOT HAVE_FIONBIO AND
		NOT HAVE_IOCTLSOCKET AND
		NOT HAVE_IOCTLSOCKET_CASE AND
		NOT HAVE_SO_NONBLOCK)
	SET(HAVE_DISABLED_NONBLOCKING TRUE)
ELSE()
	SET(HAVE_DISABLED_NONBLOCKING FALSE)
ENDIF()

##################################################################################################################
## configure

FILE(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/_include_/cmake_generated")
INCLUDE_DIRECTORIES("${CMAKE_CURRENT_BINARY_DIR}/_include_")
CONFIGURE_FILE(src/libssh2_config.h.in
	"${CMAKE_CURRENT_BINARY_DIR}/_include_/cmake_generated/libssh2_config.h")

##################################################################################################################
## compile

ADD_LIBRARY(libssh2 STATIC EXCLUDE_FROM_ALL
	"${CMAKE_CURRENT_BINARY_DIR}/_include_/cmake_generated/libssh2_config.h"
	include/libssh2.h
	include/libssh2_publickey.h
	include/libssh2_sftp.h
	src/agent.c
	src/channel.c
	src/channel.h
	src/comp.c
	src/comp.h
	src/crypt.c
	src/crypto.h
	src/global.c
	src/hostkey.c
	src/keepalive.c
	src/kex.c
	src/knownhost.c
	src/libgcrypt.h
	src/libssh2_config.h
	src/libssh2_priv.h
	src/mac.c
	src/mac.h
	src/misc.c
	src/misc.h
	src/openssl.c
	src/openssl.h
	src/packet.c
	src/packet.h
	src/publickey.c
	src/scp.c
	src/session.c
	src/session.h
	src/sftp.c
	src/sftp.h
	src/transport.c
	src/transport.h
	src/userauth.c
	src/userauth.h
	src/version.c
)

SET_TARGET_PROPERTIES(libssh2 PROPERTIES OUTPUT_NAME "ssh2")

IF(ZLIB_FOUND)
	TARGET_LINK_LIBRARIES(libssh2 ${ZLIB_LIBRARIES})
ENDIF()

TARGET_LINK_LIBRARIES(libssh2 ${OPENSSL_LIBRARIES})
