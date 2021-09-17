# What is DOL C-Kit?
DOL C-Kit is a toolkit for compiling C code (or assembly) using DevkitPPC to inject into a GameCube/Wii \*.dol executable.  It has been written in such a way that it can be adapted to many different games.  You will need [Python 3](https://www.python.org/downloads/) and [DevKitPPC](https://devkitpro.org/wiki/Getting_Started) installed to use it.  As well, DOL C-Kit is dependent on [pyelftools](https://github.com/eliben/pyelftools), JoshuaMK's fork of [dolreader](https://github.com/JoshuaMKW/dolreader), and [geckocode-libs](https://github.com/JoshuaMKW/geckocode-libs).

Credit to Yoshi2 for creating the original GC C-Kit.  DOL C-Kit couldn't exist without it.

# How to use it / The Project Class
DOL C-Kit is a Python module.  To install on Windows, run INSTALL.bat as administrator.  To install on Linux, run INSTALL.sh as superuser.

The Project class automates the tedious parts of compiling, linking, and injecting custom code into a \*.dol executable.  The Project class constructor has two optional parameters to set the "base_addr" and "verbose" member variables.  By default, they are set to None and False, respectively.

The Project class has many member variables that may be directly modified:
* src_dir: Path to C code (or assembly) source files.  Default is "".
* obj_dir: Path to output \*.o files and other files generated by DOL C-Kit to.  Default is "".
* devkitppc_path: Change this if DevKitPPC is not installed at its default location.
* project_name: Name used for certain files generated by DOL C-Kit.  Default is "project".
* gcc_flags: Non-crucial flags passed to powerpc-eabi-gcc.  Defaults include "-w", "-std=c99", "-O1", and "-fno-asynchronous-unwind-tables".
* as_flags: Non-crucial flags passed to powerpc-eabi-as.  Defaults include "-w".
* ld_flags: Non-crucial flags passed to powerpc-eabi-ld.  Defaults include nothing.
* base_addr: The location new data will be put at.  This is set by the constructor, but may be modified directly as well.
* sda_base: The value used for the \_SDA\_BASE\_ symbol.  This is set by the set\_sda\_bases method, but may be modified directly as well.
* sda2_base: The value used for the \_SDA2\_BASE\_ symbol.  This is set by the set\_sda\_bases method, but may be modified directly as well.
* verbose: Flag for additional information printing.  This is set by the constructor, but may be modified directly as well.

By shifting forward the stack, db_stack, and OSArenaLo, space for new data can be allocated.  To do this, a patching function modifying a given game's "\_\_init_registers", "OSInit", and "\_\_OSThreadInit" functions must be written.  The project's save_dol function passes two parameters to this patching function: a DolFile class, and the base_addr of your project.

## Step 1: Populate the project
* `add_c_file(filepath, gcc_flags=(), use_global_flags=True)`<br>
Add a C source file to the project.  Two optional arguments may be given: gcc_flags is a tuple of strings passed to powerpc-eabi-gcc as flags, and use_global_flags determines if the gcc_flags member of the Project class are used for this source file.

* `add_asm_file(filepath, as_flags=(), use_global_flags=True)`<br>
Add an assembly source file to the project.  Two optional arguments may be given: as_flags is a tuple of strings passed to powerpc-eabi-as as flags, and use_global_flags determines if the as_flags member of the Project class are used for this source file.

* `add_obj_file(filepath, do_cleanup=False)`<br>
Add an unlinked object file to the project.  This object file must be in the obj_dir, not the src_dir.  The cleanup method WILL DELETE FILES added by the add_obj_file method if the optional do_cleanup argument is True.

* `add_linker_script_file(filepath)`<br>
Add a [linker script file](https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_chapter/ld_3.html) to the project.  This is useful for defining symbols.

* `add_gecko_txt_file(filepath)`<br>
Add a textual [Gecko Code List](http://codes.rc24.xyz/) to the project.  When build_dol is used, codetypes 00, 02, 04, 06, 08, C6, C2, and F2 are permanently patched into the DOL.  When build_gecko is used, all given Gecko Codes are copied into a new Gecko Code List.

* `add_gecko_gct_file(filepath)`<br>
Add a binary [Gecko Code Table](http://codes.rc24.xyz/) to the project.  When build_dol is used, codetypes 00, 02, 04, 06, 08, C6, C2, and F2 are permanently patched into the DOL.  When build_gecko is used, all given Gecko Codes are copied into a new Gecko Code List.

* `hook_branch(addr, sym_name, LK=False)`<br>
Declare a branch to a symbol to be written at a given address.  Optionally, pass LK=True to declare a branchlink.

* `hook_branchlink(addr, sym_name)`<br>
Declare a branchlink to a symbol to be written at a given address.

* `hook_pointer(addr, sym_name)`<br>
Declare a pointer to a symbol to be written at a given address.

* `hook_string(addr, string, encoding = "ascii", max_strlen = None)`<br>
Declare a string to be written at a given address.  Optionally, an encoding and maximum size (in bytes) can be specified.

* `hook_file(addr, filepath, start=0, end=None, max_size = None)`<br>
Declare a file to be written at a given address.  Optionally, you may provide a start and end offset to only include a portion of the file, and a maximum size (in bytes) can be specified.  Use a negative end offset if you want the offset to be relative to the end of the file.  If the file cannot be opened, nothing is written at the given address.  This is useful for editing files embedded in the DOL.

* `hook_immediate16(addr, sym_name, modifier)`<br>
Declare a 16-bit immediate to be written at a given address.  This is useful for modifying the SIMM, UIMM, and d fields of certain instructions.  Valid modifiers include "@h", "@l", "@ha", "@sda", and "@sda2".  Make sure to use the set\_sda\_base method before trying to use the "@sda" or "sda2" modifiers.

* `hook_immediate12(addr, w, i, sym_name, modifier)`<br>
Same thing as add\_immediate\_16, but for the 12-bit immediate field of Paired-Singles load/store instructions.  The w and i fields of the original instruction must also be provided.

* `set_osarena_patcher(function)`<br>
Give your project a game-specific patching function to use to allocate space for new data.

* `set_sda_bases(sda_base, sda2_base)`<br>
Set the \_SDA\_BASE\_ and \_SDA2\_BASE\_ symbols.  These values get passed to the linker.  They are also important for the @sda and @sda2 modifiers for Immediate16Hooks.

## Step 2: Build the project
* `build_dol(in_dol_path, out_dol_path)`<br>
Compile, assemble, and link all source files, hooks, and supported Gecko Codes into a \*.dol executable.  If no base_addr is specified, the ROM end will automatically be detected and used.  A new text section will be allocated to contain the new data.  If no text sections are available, a data section will be allocated instead.<br>
Note: Automatic ROM end detection does not work for DOLs that allocate space for .sbss2.

* `build_gecko(gecko_path)`<br>
Compile, assemble, and link all source files, hooks, and Gecko Codes into a large Gecko Code List.  OSArenaLo patchers are not used, and likely never will be worth implementing be due to timing limitations of Gecko Codes.  Instead, existing data must be overwritten.

* `save_map(map_path)`<br>
Generate a CodeWarrior-like symbol map from the project.  Run this after building but before cleanup.

* `cleanup()`<br>
Delete unimportant files created by DOL C-Kit.  This includes unlinked \*.o files, and <project_name>.o, <project_name>.bin, and <project_name>.map.
