# pico-littlefs
To be included as project submodule.

A small C/C++ Posix like journaling file system for the Raspberry Pico using a size configurable
portion of its SPI flash. Adapted from the [little-fs ARM project](https://github.com/littlefs-project/littlefs.git).

Pertinent define near the top of file `pico_hal.c` determines the size of the flash file system
located to the top of flash.

```
#define FS_SIZE ((256 * 1024) - (FS_RESERVE_AT_END))
```
## Pico W Support
If you use the Pico W for Bluetooth, the bonded device list is stored in the
Pico W board's 2M flash chip at the end of flash memory. This will conflict
with the flash memory pico-littlefs uses for non-volatile storage. To avoid
the conflict, the file `pico_hal.c` contains some preprocessor defines as follows:
```
#if defined(RPPICOMIDI_PICO_W) && RPPICOMIDI_PICO_W
// <snip comments>
#define FS_RESERVE_AT_END ((FLASH_SECTOR_SIZE)*2)
#else
#define FS_RESERVE_AT_END 0
#endif
#define FS_SIZE ((256 * 1024) - (FS_RESERVE_AT_END))
```

Make sure to add
```
RPPICOMIDI_PICO_W=1
```
to the `target_compile_definitions` in your project's `CMakeLists.txt`.
For more details, see the comments in `pico_hal.c`.

## Pico Multi-Core Support
If your code is only running from a single core, checkout the git branch called
"main" instead. The "multicore-lockout" branch is intended for programs where
one core performs file system access and the other core does not.

The Pico board's RP2040 chip normally uses XIP to run code directly from flash.
Writing to flash while code is running from flash will cause undefined behavior.
The `flash_range_erase()` and the `flash_range_program()` functions in the Pico
SDK will run from on-chip SRAM. In a single-core program, that is sufficient.
If both cores are running and using XIP, though, the core that is not using the
file system needs to be completely halted with interrupts disabled before
either of these `flash_range_` functions execute. The code in this git branch
uses the Pico SDK's multicore\_lockout API to accomplish this. To use this API,
you have to set up the core that does not do file access as a multicore\_lockout
"victim." The following example show how to start up code with core 0 doing file
access and core 1 not doing file access. The main program on core 1 is called
`core1_main` and the main program on core 0 is just called main(). Blocks of
code marked `<snip>` are omitted because they differ for every application.

```
// core1: handle host events
void core1_main()
{
	// perform some initialization
	<snip>

    multicore_lockout_victim_init(); // need to lockout core 1 when core 0 writes to flash

    // rest of the code in core1_main()
	<snip>
}

int main()
{
	// perform some initialization
	<snip>
	
    // Reset core 1 and launch core 1 program
    multicore_reset_core1();    
    multicore_launch_core1(core1_main);

	// rest of the code in main() for core 0
	<snip>
```

## Functions

```
// Mounts a file system
//
// Optionally formats a new file system.
//
// Returns a negative error code on failure.
int pico_mount(bool format);

// Unmounts a file system
//
// Returns a negative error code on failure.
int pico_unmount(void);

// Removes a file or directory
//
// If removing a directory, the directory must be empty.
// Returns a negative error code on failure.
int pico_remove(const char* path);

// Open a file
//
// The mode that the file is opened in is determined by the flags, which
// are values from the enum lfs_open_flags that are bitwise-ored together.
//
// Returns opened file handle. Returns a negative error code on failure.
int pico_open(const char* path, int flags);

// Close a file
//
// Any pending writes are written out to storage as though
// sync had been called and releases any allocated resources.
//
// Returns a negative error code on failure.
int pico_close(int file);

// Return file system statistics
//
// Fills out the pico_fsstat_t structure, based on the specified file or
// directory. Returns a negative error code on failure.
int pico_fsstat(struct pico_fsstat_t* stat);

// Change the position of the file to the beginning of the file
//
// Equivalent to pico_lseek(lfs, file, 0, LFS_SEEK_SET)
// Returns a negative error code on failure.
int pico_rewind(int file);

// Rename or move a file or directory
//
// If the destination exists, it must match the source in type.
// If the destination is a directory, the directory must be empty.
//
// Returns a negative error code on failure.
int pico_rename(const char* oldpath, const char* newpath);

// Read data from file
//
// Takes a buffer and size indicating where to store the read data.
// Returns the number of bytes read, or a negative error code on failure.
lfs_size_t pico_read(int file, void* buffer, lfs_size_t size);

// Write data to file
//
// Takes a buffer and size indicating the data to write. The file will not
// actually be updated on the storage until either sync or close is called.
//
// Returns the number of bytes written, or a negative error code on failure.
lfs_size_t pico_write(int file, const void* buffer, lfs_size_t size);

// Change the position of the file
//
// The change in position is determined by the offset and whence flag.
// Returns the new position of the file, or a negative error code on failure.
lfs_soff_t pico_lseek(int file, lfs_soff_t off, int whence);

// Truncates the size of the file to the specified size
//
// Returns a negative error code on failure.
int pico_truncate(int file, lfs_off_t size);

// Return the position of the file
//
// Equivalent to pico_lseek(file, 0, LFS_SEEK_CUR)
// Returns the position of the file, or a negative error code on failure.
lfs_soff_t pico_tell(int file);

// Find info about a file or directory
//
// Fills out the info structure, based on the specified file or directory.
// Returns a negative error code on failure.
int pico_stat(const char* path, struct lfs_info* info);

// Get a custom attribute
//
// Custom attributes are uniquely identified by an 8-bit type and limited
// to LFS_ATTR_MAX bytes. When read, if the stored attribute is smaller than
// the buffer, it will be padded with zeros. If the stored attribute is larger,
// then it will be silently truncated. If no attribute is found, the error
// LFS_ERR_NOATTR is returned and the buffer is filled with zeros.
//
// Returns the size of the attribute, or a negative error code on failure.
// Note, the returned size is the size of the attribute on disk, irrespective
// of the size of the buffer. This can be used to dynamically allocate a buffer
// or check for existance.
lfs_ssize_t pico_getattr(const char* path, uint8_t type, void* buffer, lfs_size_t size);

// Set custom attributes
//
// Custom attributes are uniquely identified by an 8-bit type and limited
// to LFS_ATTR_MAX bytes. If an attribute is not found, it will be
// implicitly created.
//
// Returns a negative error code on failure.
int pico_setattr(const char* path, uint8_t type, const void* buffer, lfs_size_t size);

// Removes a custom attribute
//
// If an attribute is not found, nothing happens.
//
// Returns a negative error code on failure.
int pico_removeattr(const char* path, uint8_t type);

// Open a file with extra configuration
//
// The mode that the file is opened in is determined by the flags, which
// are values from the enum lfs_open_flags that are bitwise-ored together.
//
// The config struct provides additional config options per file as described
// above. The config struct must be allocated while the file is open, and the
// config struct must be zeroed for defaults and backwards compatibility.
//
// Returns a negative error code on failure.
int pico_opencfg(int file, const char* path, int flags, const struct lfs_file_config* config);

// Synchronize a file and storage
//
// Any pending writes are written out to storage.
// Returns a negative error code on failure.
int pico_fflush(int file);

// Return the size of the file
//
// Similar to pico_lseek(file, 0, LFS_SEEK_END)
// Returns the size of the file, or a negative error code on failure.
lfs_soff_t pico_size(int file);

// Create a directory
//
// Returns a negative error code on failure.
int pico_mkdir(const char* path);

// Open a directory
//
// Once open a directory can be used with read to iterate over files.
// Returns a negative error code on failure.
int pico_dir_open(int dir, const char* path);

// Close a directory
//
// Releases any allocated resources.
// Returns a negative error code on failure.
int pico_dir_close(int dir);

// Read an entry in the directory
//
// Fills out the info structure, based on the specified file or directory.
// Returns a positive value on success, 0 at the end of directory,
// or a negative error code on failure.
int pico_dir_read(int dir, struct lfs_info* info);

// Change the position of the directory
//
// The new off must be a value previous returned from tell and specifies
// an absolute offset in the directory seek.
//
// Returns a negative error code on failure.
int pico_dir_seek(int dir, lfs_off_t off);

// Return the position of the directory
//
// The returned offset is only meant to be consumed by seek and may not make
// sense, but does indicate the current position in the directory iteration.
//
// Returns the position of the directory, or a negative error code on failure.
lfs_soff_t pico_dir_tell(int dir);

// Change the position of the directory to the beginning of the directory
//
// Returns a negative error code on failure.
int pico_dir_rewind(int dir);

// Return pointer to string representation of error code.
//
// Returns a negative error code on failure.
const char* pico_errmsg(int err);

```

