# Set up a C and C++ project with useful defaults for debugging
project(
  'slipstream',
  ['c', 'cpp'],
  version: '0.1.0',
  license: 'Apache-2.0',
  meson_version: '>=1.0.0',
  default_options: [
    'buildtype=debugoptimized',
    'cpp_std=gnu++11',
    'c_std=gnu11',
    'default_library=shared',
    'warning_level=everything',
    'b_lto=false',
  ],
)

# Aliases to significant locations in the source tree
src = 'src'
extern = 'extern'
lua_resty = extern / 'lua-resty-base-encoding'
quick_args = extern / 'quick_arg_parser'
spcdns = extern / 'SPCDNS' / src

# Sources common to both client and server
common_sources = files(
  src / 'slipstream_sockloop.c',
  src / 'slipstream_utils.c',
  src / 'slipstream_inline_dots.c',
  lua_resty / 'base32.c',
  spcdns / 'codec.c',
  spcdns / 'mappings.c',
  spcdns / 'netsimple.c',
  spcdns / 'output.c',
)

# Various directories with headers for the sources
include_dirs = include_directories('include', extern, quick_args, spcdns, lua_resty)

# Default linker arguments
link_args = []

# Set up static links — flags differ between GCC toolchains and the Android NDK (clang)
is_static = get_option('default_library') == 'static'
if is_static and host_machine.system() != 'darwin'
  if host_machine.system() == 'android'
    # NDK clang uses libc++; -static-libgcc is a GCC-only flag and must not be used here
    link_args = ['-static-c++']
  else
    link_args = [
      '-static-libgcc',
      '-static-libstdc++',
    ]
  endif
endif

# Import the CMake module
cmake = import('cmake')

# Get the cmake options objects
picoquic_opts = cmake.subproject_options()

# Add necessary CMake defines
picoquic_opts.add_cmake_defines(
  [
    {'PICOQUIC_FETCH_PTLS': 'ON'},
    {'BUILD_HTTP': 'OFF'},
    {'ENABLE_ASAN': 'OFF'},
    {'ENABLE_UBSAN': 'OFF'},
    {'CMAKE_POLICY_VERSION_MINIMUM': '3.5'},
    {'BUILD_DEMO': 'OFF'},
    {'BUILD_TESTING': 'OFF'},
  ],
)
# Configure picoquic debug printing
if get_option('buildtype').startswith('debug')
  picoquic_opts.add_cmake_defines(
    {'DISABLE_DEBUG_PRINTF': 'OFF'},
  )
else
  picoquic_opts.add_cmake_defines(
    {'DISABLE_DEBUG_PRINTF': 'ON'},
  )
endif
# Configure optional picoquic logging
loglib_enabled = get_option('build_loglib')
if loglib_enabled
  picoquic_opts.add_cmake_defines(
    {'BUILD_LOGREADER': 'ON'},
  )
  add_project_arguments('-DBUILD_LOGLIB', language: ['c', 'cpp'])
else
  picoquic_opts.add_cmake_defines(
    {'BUILD_LOGREADER': 'OFF'},
  )
endif

# When cross-compiling for Android, propagate the NDK toolchain and OpenSSL
# paths to picoquic's inner CMake build via cmake.subproject_options().
# Values come from Meson user options set on the meson setup command line:
#   -Dandroid_ndk_root=/opt/android-ndk
#   -Dandroid_openssl_prefix=/opt/openssl-android
#   -Dandroid_api=34
if host_machine.system() == 'android'
  android_ndk = get_option('android_ndk_root')
  android_ssl = get_option('android_openssl_prefix')
  android_api = get_option('android_api').to_string()

  if android_ndk != ''
    picoquic_opts.add_cmake_defines([
      {'CMAKE_TOOLCHAIN_FILE': android_ndk / 'build' / 'cmake' / 'android.toolchain.cmake'},
      {'ANDROID_ABI': 'arm64-v8a'},
      {'ANDROID_PLATFORM': 'android-' + android_api},
    ])
  endif

  if android_ssl != ''
    picoquic_opts.add_cmake_defines([
      {'OPENSSL_ROOT_DIR': android_ssl},
      {'OPENSSL_INCLUDE_DIR': android_ssl / 'include'},
      {'OPENSSL_CRYPTO_LIBRARY': android_ssl / 'lib' / 'libcrypto.a'},
      {'OPENSSL_SSL_LIBRARY': android_ssl / 'lib' / 'libssl.a'},
    ])
  endif
endif

# Reference picoquic as a CMake subproject
picoquic_proj = cmake.subproject(
  'picoquic',
  options: picoquic_opts,
)

# Dependencies used by both executables
# On Android, pthreads is part of Bionic libc — no separate -lpthread link needed
if host_machine.system() == 'android'
  slipstream_cli_deps = [
    picoquic_proj.dependency('picoquic-core'),
  ]
else
  slipstream_cli_deps = [
    dependency('threads'), # for picoquic
    picoquic_proj.dependency('picoquic-core'),
  ]
endif
# Link optional picoquic logging
if loglib_enabled
  slipstream_cli_deps += picoquic_proj.dependency('picoquic-log')
endif

# Sources only used by client
client_sources = files(
  src / 'slipstream_client.c',
  src / 'slipstream_client_cli.cpp',
)

# Sources only used by server
server_sources = files(
  src / 'slipstream_server.c',
  src / 'slipstream_server_cc.c',
  src / 'slipstream_server_cli.cpp',
)

# The final client executable
executable(
  'slipstream-client',
  client_sources + common_sources,
  dependencies: slipstream_cli_deps,
  include_directories: include_dirs,
  link_args: link_args,
  build_by_default: true,
)

# The final server executable
executable(
  'slipstream-server',
  server_sources + common_sources,
  dependencies: slipstream_cli_deps,
  include_directories: include_dirs,
  link_args: link_args,
  build_by_default: true,
)
