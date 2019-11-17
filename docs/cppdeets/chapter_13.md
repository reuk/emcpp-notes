# Filesystem

C++17 introduces standard facilities for dealing with directories, paths, and file permissions.
`std::filesystem` is based on Boost's filesystem library, but with extensions to support non-POSIX
systems.

## Core Components

`namespace fs = std::filesystem;` is implied:

- `fs::path` allows manipulation of paths to files/directories that may or may not exist.
- `fs::directory_entry` represents a path which exists, with metadata about the file/directory
  such as modification time and size.
- Iterators allow (recursive or not) iteration of directory contents.

## `fs::path`

### Getters

Contains a pathname string which names a file that may or may not exist. Made up of:

- root-name (optional): this is the drive name (`C:\`) on Windows, but unnecessary on POSIX
- root-directory (optional): relative paths are relative to this directory
- Zero or more of the following:
  - filename: a sequence of non-directory-separator characters that identifies a filesystem entity
    - dot (`.`): a special filename corresponding to the current directory
    - dot-dot (`..`): a special filename corresponding to the parent directory
  - directory separator: a forward-slash (`/`) or `path::preferred_separator`

There are queries for the following components, which return an empty path if the requested
component is not present. There are also `has_`-prefixed versions which check for the existence of
a particular path component:

- `root_name`
- `root_directory`
- `root_path` (the root_name and root_directory combined)
- `relative_path`
- `parent_path`
- `filename` (the final filename component of the path if it exists)
- `stem` (the substring from the beginning of `filename()` up to but not including the final `.`)
- `extension` (from the final `.` to the end of the filename, unless the filename begins with `.`)

`path` also exposes iterators over each of the individual filename components in the path, e.g.

```cpp
for (const auto& part : fs::path { "C:/Windows/system.ini" })
    std::cout << part << '\n';
```

- `C:`
- `Windows`
- `system.ini

### Path Operations

- `append`: combine with a directory separator
- `concat`: combine without a directory separator
  - There are also overloads of operators `+=`, `/=`, `/` (but weirdly not `+`)
- `clear`: you guessed it
- `remove_filename`: removes the final filename component from the path
- `replace_filename`: creates path to a sibling file
- `replace_extension`: yeah
- `swap`: pretty obvious
- `compare`: lexical comparison, returning an integer denoting ordering/equality
- `empty`: like other containers, checks whether the container is empty

Comparison works as expected, for the most part. Shorter paths are ordered before longer paths that
are otherwise equivalent. Paths without root directories are ordered before paths with root
directories.

Paths have stream operator overloads to read/write to streams while preserving correct
(platform-dependent) formatting.

### Conversions

To get at the platform-dependent path representation, you can use `c_str()` or `native()`. This
native format can also be converted to a specific character width (e.g. u8, u16, u32), but this
should be used with care on platforms which use wide characters for pathnames (like Windows).

## `directory_entry`

Can be constructed from a `path`, throwing or updating an error code if construction fails.

Can also be created via a `directory_iterator` or `recursive_directory_iterator`. These are input
iterators, which visit directory contents in an unspecified order, and may or may not be updated if
a file is added/removed after the iterator is constructed.

Both iterator constructors actually return a range with a begin/end, suitable for use with a
range-for (or `std::begin` and `std::end` can be used to extract real iterators for use with
algorithms).

Lots of getters, all fairly self-explanatory.

### Non-member Utilities

There are some non-member functions that can query properties about a `path`, without needing to
construct a `directory_entry`. For example, you could call `fs::exists (somePath)` to check that
a particular file exists before using the path to construct a `directory_entry`.

There are also utilities like `absolute` and `relative` for adjusting the path representation,
`equivalent` to check whether two paths refer to the same file, and `current_path` to get/set the
current working directory.

Finally, there are functions to copy files/directories, create new directories and links, delete
files/directories, get temporary filepaths, compute free space on a device...

Currently, file modification times use an implementation-defined clock, which makes it difficult
to write portable code. In C++20 there should be a portable, properly-specified `file_clock`.

File permissions can be queried/written by composing/decomposing bitfields using the `fs::perms`
enum. `status` returns the current file permissions (use `operator&` to extract specific permission
bits) and `permissions` can replace/add/remove permissions on a specific file. Again, the full
suite of POSIX permissions is not available on Windows.

TODO: check whether any of this stuff is actually supported on macOS, and if so, which SDK is
required.
