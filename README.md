# LabVIEW VI-Unlocker

This tools enables reliably unlocking all types of view/edit-restricted LabVIEW files with password-protected (but present) source code, from binary virtual instrument (VI) files to XML-based library-type files (e.g., \*.lvlib or \*.lvclass). The unlcoker code is entirely written in LabVIEW (version 23) and should be cross-platform because it uses no external libraries. A simple GUI tool allows processing an entire folder with user-selectable file types. All LabVIEW file versions starting from version 6 and later are supported.

## Usage

LabVIEW version 2023 or newer is required to view, edit or run the code. You can use the free [community edition](https://www.ni.com/en/support/downloads/software-products/download.labview-community.html). Download the repository into a local folder (the main Unlock_project_folder.vi and the subvis folder) and open the Unlock_project_folder.vi front panel in LabVIEW. Select the directory with locked LabVIEW files (sub-directories are processed recursively) and the output directory and run the VI. You can edit the list of file extensions (comma separated) processed by the unlocker before running it. The tool creates unlocked copies of all files in the source directory matching the file extensions and puts them into the output directory maintaining the original folder structure. Warning: If the source and destination directories are identical, the source files will be replaced with the unlocked versions. Unlocking a single file is also possible by running the VI_unlock.vi in the subvis folder directly, but it can only unlock binary RSRC type files (e.g., VIs). It also allows rebuilding broken VI hashes by recreating them from the VI file contents (disable salt verification in that case). The tool supports binary file versions 6 and later. Even file versions later than the LabVIEW version running the tool are theoretically supported as long as the lock-related file components are not changed in the future (e.g., the tool running with LabVIEW 2023 can unlock files created with LabVIEW 2024).

The delete_license.vi is a small GUI tool to list and (selectively) delete the registration information of installed third-party add-ons from the Windows registry across all LabVIEW installations (run as administrator).

## Shoulders of giants and so on

This tool is based on tremendous reverse-engineering efforts of the VI file format, carrierd out over many years by several enthusiasts. In particular, the [VI-Explorer](https://github.com/tomsoftware/VI-Explorer/tree/master) that (to my knowledge) started it all, [VI-Hacker](https://github.com/rcpacini/LabVIEW-VI-Hacker) expanded on the documentation, and [pylabview](https://github.com/mefistotelis/pylabview) is by far the most extensive open-source library for parsing the VI file format. The documentation on the binary resource block data responsible for the password hashing and data type descriptions was crucial for this project. Besides rewriting the code in a more compact and performant way, I added some improvements and expansions:
- Use LabVIEW's built-in (but undocumented) inflate function (ZUncompressLStr) to uncompress deflated binary blocks instead of using an external zlib.dll
- Use LabVIEW's (also undocumented) MD5 hash function in C (GetMD5Digest), which is about 10 times faster than the G-code implementation
- The hash salt is properly calculated from the VCTP data block (terminal and type definitions) instead of bruteforce hashing, which allows handling edge cases with a huge number of connector pane controls beyond the bruteforce search space
- Full support for unlocking library member files and XML-type files such as *.lvclass with embedded binary controls
- Read library icons as emedded binary objects

## How it works

Binary LabVIEW files (such as VIs) follow the ancient Apple RSRC file format, which is a collection/database of proprietary binary blocks of data with 4-character names. Different blocks store file components such as the front panel, the block diagram, icons, but also metadata like flags and data type descriptors (e.g. numeric, string, array, cluster etc.). The BDPW block contains 3 MD5 hashes: the block diagram password hash (even for an empty password), and two more hashes called Hash1 and Hash2. Hash1 is calculated from the concatenation of the password hash with the list of owning libraries (colon separated), the LVSR (LabVIEW save record) block, and a 12-byte VI-specific salt; the salt was added in LabVIEW version 12. Hash2 is calculated from a concatenation of Hash1 with the uncompressed block diagram heap BDHx (x denotes different possible block diagram formats, b, c, P, X, T; b and c are compressed formats); Hash2 was added in LabVIEW version 8.2. The salt is the number of numeric, string and path controls on the VI's connector pane, including (recursively!) inside of containers (arrays, clusters, typedefs, refnums, terminals, sets and maps). The 3 counts (in the order numerics, strings, paths) are expressed as 32-bit integers in little-endian format and concatenated into a 12-byte string. Upon loading the VI into memory, LabVIEW calculates the Hash1 and Hash2 hashes and compares them with the ones stored in the BDPW block - if they don't match, LabVIEW refuses to open the VI. If they match, the VI opens but you have to enter the correct password (unless it's blank) in order to view or edit the block diagram. In addition, if the VI's owning library is password-protected, the library password's MD5 hash is stored in the VI's LVSR block together with a "locked" flag and the library password has to be entered in order to view the source code.

In order to unlock a VI, the BDPW block has to be overwritten with an empty password hash and re-calcuated Hash1 and Hash2 using the new (empty) password hash and the other block data. The salt is the most challenging component to calculate because it requires parsing potentially complex nested data types, which are stored in the VCTP (VI consolidated types) block. The block stores all data type descriptors used in the VI, and container types (arrays, clusters etc.) reference other type descriptors by index. The VI's connector pane, which is the source of the salt, has the type "terminal" (also called "function") and contains a list of client type descriptors (i.e., controls on the connector pane). This terminal type definition is also used for other terminals inside the VI (e.g., subVIs or strictly typed VI references). In addition, the VCTP block stores a list of type descriptors (by index) which are used in other sections of the VI, such as on the front panel or block diagram. This list also contains the VI's connector pane terminal, which is referenced by the index number stored in the VI's CONP block. Therefore, the CONP block uniquely identifies the terminal used for the salt calculation in the VCTP block. For polymorphic VIs the CONP block points to a "polyVI" type definition without any clients, since such a VI has no connector pane and therefore always a salt of 0 (12 bytes).

Alternatively, using VI-server scripting functions to read the VI file with LabVIEW (open VI reference) and count the connector pane controls via VI class properties is relatively easy but this method is not feasible for large libraries because it tries to load all dependencies of the VI into memory. In addition, some scripting functions are not available for VIs inside locked libraries and will fail to open a reference.

Directly bruteforcing the 3 numbers constituting the salt is a much simpler way in almost all cases, but it is not guaranteed to work in some edge cases with very large numbers of controls which might exceed the search space limit (type descriptors have at least a 16-bit address space). The bruteforce method with a search space of up to 255 for each of the 3 numbers is still used here as a fallback if the type descriptor counting method (parsing the VCTP block) fails. All VIs in LabVIEW's vi.lib directory can be successfully processed with the VCTP parsing method without a fallback on bruteforce. In addition, if the VI is a member of a password-protected library, the library password hash in the LVSR block has to be overwritten with an empty password hash and the "locked" flag (bit) has to be removed.

Unlocking XML-based library files is somewhat trivial, simply by deleting the (cleartext) entries and signatures related to the password hash. Some XML-based file types such as class objects (\*.lvclass) contain a control RSRC file (\*.ctl) as an embedded binary string (XML Type="Bin"). If the class object is a member of a password-protected library, then the embedded control contains a library password hash inside its LVSR block, which needs to be overwritten with a blank password hash to unlock the class object. The embedded binary string uses a proprietary base64 encoding with consecutive ASCII values starting at 0x21 ("!") with escaped XML characters (&, <, >). This string represents a flattened variant containing a byte array, which is the bytestream of the embedded control RSRC file. After modifying the control (as a bytestream), the flattening/encoding/escaping is simply applied in reverse. The same emedded binary string format is used to encode the icon of the XML-based file. Here, the resulting bytestream represents a deflated flattened LabVIEW pixmap.

## What about the copyright police?

The copyright law has a very narrow scope with regards to what happens within the confines of your own hard drive. The tool is not circumventing any copy protection (because there is none) or breaking encryption (which would only be relevant in connection with the former point), therefore simply removing or changing a VI password hash has no conceivable legal implications. Whatever happens beyond that is outside the scope or functionality of the tool. But since I probably have to spell out the obvious anyway: You are responsible to uphold any license agreements you have voluntarily entered into. Any information provided here is solely for research purposes.

All components of a VI which could be considered intellectual property (such as the block diagram code) are unencrypted and freely accessible in the file regardless of the password hash. The "protection" arises solely from the difficulty to read/edit the block diagram or front panel contents with an alternative editor. For some reason National Instruments decided to market this fact for "licensing" the mere viewing of source code, which is unheard of in any other programming language. The lack of actual source code encryption is by design in order to preserve the ability to recompile the code with a different LabVIEW version - if this is not required, the source code can be entirely removed from the VI file leaving only the compiled code. Hence, the password protection provides some "security" through obscurity, but can never enforce a license. Only lawyers can.
