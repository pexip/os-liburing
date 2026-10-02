# Downstream meson build for liburing, used by the pexip media stack.
#
# Upstream builds with ./configure + make, which the pexbuild chain has no
# generator for -- every other external dependency is meson or cmake. This
# builds the same library from the same sources, and is kept deliberately
# narrow: only the library is built, not liburing-ffi (which exists for
# languages that cannot use the static inline helpers in liburing.h), and not
# the tests or examples.
#
# configure's generated headers are reproduced here as templates, but are
# written into the build directory rather than the source tree:
#
#   src/include/liburing/io_uring_version.h  from the version in liburing.spec
#   src/include/liburing/compat.h            from feature probes, for kernel
#                                            headers older than the API
#                                            liburing exposes
#
# configure's third output, config-host.h, is not needed: of everything it
# defines, src/ only ever consults CONFIG_NOLIBC and CONFIG_USE_SANITIZER, and
# this build wants neither. CONFIG_NOLIBC swaps libc's syscall() for liburing's
# own stubs; it is a small win that costs -nostdlib on a library linked into
# every build, and it only covers four architectures.
#
# The probes and their fallback blocks are configure's verbatim, so the
# generated headers match what configure would have produced.

project('liburing', 'c',
  version : '0',
  meson_version : '>= 1.1',
  license : 'MIT',
  default_options : [
    'c_std=gnu11',
    'warning_level=1',
    'werror=false',
  ],
)

if host_machine.system() != 'linux'
  error('liburing is Linux only')
endif

fs = import('fs')
cc = meson.get_compiler('c')
pkgconfig = import('pkgconfig')

# configure compiles its probes with -D_GNU_SOURCE (see CFLAGS in its
# configure script); several of the types below are only declared under it.
liburing_probe_args = ['-D_GNU_SOURCE']

liburing_src = 'src'

# Upstream's Makefile.common shells out to awk over liburing.spec for the
# version, and configure turns that into io_uring_version.h. Same source of
# truth here, so a submodule bump is picked up without editing this file.
liburing_version = ''
foreach line : fs.read('liburing.spec').split('\n')
  if line.startswith('Version:')
    liburing_version = line.split(':')[1].strip()
    break
  endif
endforeach
assert(liburing_version != '',
  'could not read "Version:" from liburing.spec')

version_arr = liburing_version.split('.')
version_conf = configuration_data()
version_conf.set('VERSION_MAJOR', version_arr[0])
version_conf.set('VERSION_MINOR', version_arr[1])

# compat.h. Each probe below is configure's test program verbatim; the
# fallback text is the block configure emits when the probe fails.
compat_conf = configuration_data()

have_kernel_rwf_t = cc.compiles('''
#include <linux/fs.h>
int main(int argc, char **argv)
{
  __kernel_rwf_t x;
  x = 0;
  return x;
}
''', name : '__kernel_rwf_t', args : liburing_probe_args)
compat_conf.set('KERNEL_RWF_T',
  have_kernel_rwf_t ? '' : 'typedef int __kernel_rwf_t;\n')

have_kernel_timespec = cc.compiles('''
#include <linux/time.h>
#include <linux/time_types.h>
int main(int argc, char **argv)
{
  struct __kernel_timespec ts;
  ts.tv_sec = 0;
  ts.tv_nsec = 1;
  return 0;
}
''', name : 'struct __kernel_timespec', args : liburing_probe_args)
if have_kernel_timespec
  compat_conf.set('KERNEL_TIMESPEC', '''#include <linux/time_types.h>
/* <linux/time_types.h> is included above and not needed again */
#define UAPI_LINUX_IO_URING_H_SKIP_LINUX_TIME_TYPES_H 1
''')
else
  compat_conf.set('KERNEL_TIMESPEC', '''#include <stdint.h>

struct __kernel_timespec {
	int64_t		tv_sec;
	long long	tv_nsec;
};

/* <linux/time_types.h> is not available, so it can't be included */
#define UAPI_LINUX_IO_URING_H_SKIP_LINUX_TIME_TYPES_H 1
''')
endif

have_open_how = cc.compiles('''
#include <sys/types.h>
#include <fcntl.h>
#include <string.h>
#include <linux/openat2.h>
int main(int argc, char **argv)
{
  struct open_how how;
  how.flags = 0;
  how.mode = 0;
  how.resolve = 0;
  return 0;
}
''', name : 'struct open_how', args : liburing_probe_args)
if have_open_how
  compat_conf.set('OPEN_HOW', '#include <linux/openat2.h>\n')
else
  compat_conf.set('OPEN_HOW', '''#include <inttypes.h>

struct open_how {
	uint64_t	flags;
	uint64_t	mode;
	uint64_t	resolve;
};
''')
endif

statx_prog = '''
#include <sys/types.h>
@0@#include <unistd.h>
#include <fcntl.h>
#include <string.h>
@1@int main(int argc, char **argv)
{
  struct statx x;

  return memset(&x, 0, sizeof(x)) != NULL;
}
'''
have_statx = cc.compiles(statx_prog.format('#include <sys/stat.h>\n', ''),
  name : 'struct statx', args : liburing_probe_args)
have_glibc_statx = cc.compiles(statx_prog.format('', '#include <sys/stat.h>\n'),
  name : 'struct statx in glibc', args : liburing_probe_args)
# Only pulled in when <sys/stat.h> has to come before the other headers to
# declare it; when glibc declares it either way, liburing.h's own include
# of <sys/stat.h> is enough.
compat_conf.set('STATX',
  (have_statx and not have_glibc_statx) ? '#include <sys/stat.h>\n' : '')

have_futexv = cc.compiles('''
#include <linux/futex.h>
#include <unistd.h>
#include <string.h>
int main(void)
{
  struct futex_waitv fw;

  memset(&fw, FUTEX_32, sizeof(fw));

  return sizeof(struct futex_waitv);
}
''', name : 'struct futex_waitv', args : liburing_probe_args)
if have_futexv
  compat_conf.set('FUTEXV', '')
else
  compat_conf.set('FUTEXV', '''#include <inttypes.h>

#define FUTEX_32	2
#define FUTEX_WAITV_MAX	128

struct futex_waitv {
	uint64_t	val;
	uint64_t	uaddr;
	uint32_t	flags;
	uint32_t	__reserved;
};
''')
endif

have_idtype_t = cc.compiles('''
#include <sys/wait.h>
int main(void)
{
  idtype_t v;
  return 0;
}
''', name : 'idtype_t', args : liburing_probe_args)
if have_idtype_t
  compat_conf.set('IDTYPE_T', '')
else
  compat_conf.set('IDTYPE_T', '''typedef enum
{
  P_ALL,		/* Wait for any child.  */
  P_PID,		/* Wait for specified process.  */
  P_PGID		/* Wait for members of process group.  */
} idtype_t;
''')
endif

have_discard_cmd = cc.compiles('''
#include <linux/blkdev.h>
int main(void)
{
  return BLOCK_URING_CMD_DISCARD;
}
''', name : 'BLOCK_URING_CMD_DISCARD', args : liburing_probe_args)
if have_discard_cmd
  compat_conf.set('DISCARD_CMD', '#include <linux/blkdev.h>\n')
else
  compat_conf.set('DISCARD_CMD', '''#include <linux/ioctl.h>

#ifndef BLOCK_URING_CMD_DISCARD
#define BLOCK_URING_CMD_DISCARD                        _IO(0x12, 0)
#endif
''')
endif

# Generated into the build tree at the same path configure would have written
# them to in the source tree, so include_directories('src/include') -- which
# covers both the source and build copies -- resolves them unchanged.
subdir('src/include/liburing')

liburing_sources = files(
  liburing_src / 'setup.c',
  liburing_src / 'queue.c',
  liburing_src / 'register.c',
  liburing_src / 'syscall.c',
  liburing_src / 'version.c',
)

liburing_inc = include_directories('src/include')

liburing_c_args = [
  '-D_GNU_SOURCE',
  '-D_LARGEFILE_SOURCE',
  '-D_FILE_OFFSET_BITS=64',
  '-DLIBURING_INTERNAL',
  '-Wno-unused-parameter',
]

# Upstream links the shared library against its own version script, so match
# that. It does two things: it hides four internal symbols (io_uring_mmap,
# io_uring_unmap_rings, io_uring_setup_ring_pointers and
# __io_uring_queue_init_params), and -- the reason it matters -- it stamps the
# remaining exports with the LIBURING_2.x node they belong to, so consumers
# link against the same versioned ABI they would get from a distro liburing.
# Absolute: the linker runs from the build directory, and a relative path
# would make has_link_argument() fail and silently drop the version script.
liburing_map = meson.current_source_dir() / liburing_src / 'liburing.map'
liburing_link_args = []
liburing_link_depends = []
vscript_arg = '-Wl,--version-script=' + liburing_map
if cc.has_link_argument(vscript_arg)
  liburing_link_args += vscript_arg
  liburing_link_depends += files(liburing_map)
endif

liburing_lib = library('uring',
  liburing_sources,
  liburing_generated_headers,
  c_args : liburing_c_args,
  include_directories : liburing_inc,
  link_args : liburing_link_args,
  link_depends : liburing_link_depends,
  version : liburing_version,
  soversion : version_arr[0],
  install : true,
)

install_headers(liburing_src / 'include' / 'liburing.h')
install_headers(
  liburing_src / 'include' / 'liburing' / 'barrier.h',
  liburing_src / 'include' / 'liburing' / 'io_uring.h',
  liburing_src / 'include' / 'liburing' / 'sanitize.h',
  subdir : 'liburing',
)

liburing_dep = declare_dependency(
  link_with : liburing_lib,
  sources : liburing_generated_headers,
  include_directories : liburing_inc,
)

pkgconfig.generate(liburing_lib,
  name : 'liburing',
  description : 'io_uring library',
  url : 'https://git.kernel.dk/cgit/liburing/',
  version : liburing_version,
)

