# configure writes these two headers into this directory in the source tree.
# Generating them here instead keeps a configured build tree from dirtying the
# checkout, while leaving the include path identical: liburing.h refers to them
# as <liburing/compat.h> and <liburing/io_uring_version.h>, and
# include_directories('src/include') covers the build copy as well as source.

liburing_generated_headers = [
  configure_file(
    input : 'io_uring_version.h.in',
    output : 'io_uring_version.h',
    configuration : version_conf,
    install : true,
    install_dir : get_option('includedir') / 'liburing',
  ),
  configure_file(
    input : 'compat.h.in',
    output : 'compat.h',
    configuration : compat_conf,
    install : true,
    install_dir : get_option('includedir') / 'liburing',
  ),
]

