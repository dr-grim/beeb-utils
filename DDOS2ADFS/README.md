# DDOS2ADFS

Is a utility to help copy multi-volume Opus DDOS 720k floppy discs to an ADFS disc - very useful for anyone with a bundle of 720k floppies that need transferring (say) to a GoTEK or similar.

Given that this script was written just for this specific use case, I could make a fair few assumptions - and could use the folder layout I wanted. So, for instance, minimising disc passes wasn't a consideration as no media had to be swapped, and the utility can be left to run unattended - hence speed was not a large consideration.

## Mapping from DDOS to ADFS

Given a multi-volume DDOS disc, files will be mapped as follows:

Opus DDOS file | ADFS file
-|-
`:0A.$.file` | `:0.$.0A.file`
`:0A.e.other` | `:0.$.0A.e.other`
`:0B.a.dead` | `:0.$.0B.a.dead`
`:2C.$.parrot` | `:0.$.2C.parrot`

This enables a double-sided disc with up to 16 sub-volumes to be stored on a single ADFS disc. Note that folders are created as required.

## Code design
The utility:
1. Scans the DDOS sectors on track 0 to discover what DDOS drive volumes are populated
2. Scans each DDOS volume catalogue to read which files are present - and extract load & execution addresses, along with file length
3. Uses `OSWORD &7F` to read sector data
4. Uses `OSGBPB 1` and `OSGBPB 3` to read and write chunks of data
5. Plain `OSCLI` to `*SAVE` as a lazy way of setting load & execute addresses
6. `OSCLI` to `*CDIR` and `*DIR` in `ADFS` as required
7. Drive volumes in DDOS are mapped to subfolders in `ADFS`, so volume `0C` maps to folder `:0.$.0A` on ADFS
8. Files in folder `$` are copied to the root of the relevant volume folder
9. All other folders have a matching subfolder created in the relevant volume folder

## Code structure

### Global variables
1. `logLevel%` changes the verbosity of output - 0=quiet, 1=medium, 2=noisy
1. File information of all files in current volume (up to 31 files) is tored in global variables: `fileNames$(30)`, `loadAddresses%(30)`, `execAddresses%(30)`, `lengths%(30)`, `directoryNames$(30)` and `fileLocks%(30)`.
1. Presence of a logical colume on the current drive is also stored in a global `volumePresent%(7)`
1. `volumeTitle$` holds title of current volume
1. `currentFS$` records current FS in use (`DISC` for DDOS, `ADFS` for ADFS)
1. `oswordScratch%` allocates 32 for parameter passing to `OSWORD`
1. `rawSectorData%` allocated &0200 bytes for 2 sectors (256 bytes each); used to hold the 2 sectors per volume catalogue or single sector for volume allocation. Each of the 2 sectors are also pointed to with globals `rawSectorA%` and `rawSectorB%` for ease of use.
1. `memoryEndAddress%` notes top of useable memory, and `memoryStartAddress%` the beginning. Set by lowering `HIMEM` to allocate memory away from `BASIC`. `maxFileSize%` notes how much memory we can use.

### Main loop
1. The code iterates over disc sides (0 and 2)
1. Within each side, `PROCreadOpusVolumes` is used to read the volumes
1. Each volume is then iterated over and (if present/allocated) copied to ADFS using `PROCcopyVolume`

### PROCreadOpusVolumes(driveNumber%)
1. Reads sector 16 from track 0, which has the volume allocation information
1. Sets `volumePresent%` accordingly

### PROCcopyVolume(driveNumber%, volumeNumber%)
1. Switches to `ADFS` and uses `PROCadfsCdir` to create an appropriate subdirectory for the current volume number (viz volume 2 of drive 0 is `0B`, so on ADFS the directory `:0.$.0B` will be created)
1. Switches back to DDOS, and reads the volume catalogue
1. Iterates over all files in catalogue
1. If a file has a directory that isn't `$`, then we create a subdirectory in ADFS to match (unless we've already done this - `directoriesCreated$` keeps tabs)
1. `numChunks%` calculated to see if we need more than 1 pass to copy the file through memory (`maxFileSize%` used)
1. Switch to DDOS (`*DISC`)
1. `PROCloadData` used to read from DDOS (whole file or a chunk); **NOTE** this was also needed as using `*LOAD` would cause ADFS to fail to initialise
1. Switch to ADFS (`*ADFS`)
1. If this is the first chunk, `*SAVE` is used to store to ADFS - as this will set execution and load addresses; otherwise `PROCsaveData` is used (inverse of `PROCloadData`)
1. Once all files have been copied from the volume, the volume's title is used to set the equivalent ADFS folder's title

### FNdriveName(drive%, volume%)
Returns a 2 character string for a DDOS volume such as `0H`.

### FNreadCatalogue(driveNumber%, volumeNumber%)
Reads the given volume catalogue into the global variables, returns number of files present.

### FNword(memory%)
Returns 16 bit value from two 8 bytes.

### FNhighProcessorWord(byte%, mask%)
Sets the top 16 bits to &FFFF is the mask indicates or &0000 otherwise.

### FNtrimTrailingSpaces(string$)
Removes trailing spaces from filenames.

### PROCreadSector(drive%, track%, sector%, numSectors%)
Reads sector data.

### FNstringFromMemory(baseAddress%, length%)
Pulls given number of ASCII characters from memory, truncates resulting string by ignoring `NULL` (ASCII 0) characters.

### PROCsetFS(fs$)
If the current file system (`currentFS$`) isn't the same, then changes to this new file system (and records the change in `currentFS$`).

### PROCadfsCdir(dir$)
Switches to ADFS then uses OSCLI to `*CDIR` the requested directory.

### FNfilteredFileName(fileName$)
Replaces any `.` with `-` to prevent interpretation as a directory and file name by ADFS, rather than just a file name.

### PROCloadData(fullFileName$, memoryAddress%, length%, fileOffset%)
Uses `OSGBPB 3` to read the requested number of bytes - enabling partial load of long files.

### PROCsaveData(fullFileName$, memoryAddress%, length%, fileOffset%)
Uses `OSGBPB 1` with a file opened with `OPENUP` to enable an existing (partial) file to be extended. The first (and possibly) only chunk is saved with `*SAVE` to create the file to create the needed execution & load addresses; this function is used to store additional data from long files that won't fit in memory.

## Unused code
There are a few unused PROCedures in there, which I've left as they may come in handy:
1. `PROCprintFileCatalogue` (and related `FNfileInfoAsString` and `FNaddressToHexString(address%)`): can pretty print the current volume's file list in a style similar to `*INFO` - useful for reporting / debugging
2. `PROCcreateSubdirsAsRequired` was used to create additional subdirectories if a DDOS filename had later "." in it - but caused issues if a file also existed with out extra "." and then clashed with the folder. Resorted to replacing "." with "-" instead elsewhere and not using this.

## Work-arounds / oddities
Any issues I encountered are covered here.

### ADFS locking up
Oddly, when using `*LOAD` in DDOS, then calling `*ADFS` to switch filing systems caused ADFS to lock solid - `ESCAPE` didn't work, had to use `BREAK`. This was circumvented by using `OPENIN` in BASIC and then `OSGBPB` to read the raw data - which has the advantage of being able to read a block of data from anywhere in the file. System tested with:

ROM | Version
-|-
Opus DDOS | 1.01
ADFS | 1.50
MOS | 3.20

### Files too large for available memory
The switch to `OSGBPB` enabled me to chunk large files - if the data wouldn't fit in memory, then it could be loaded and saved in multiple passes. `HIMEM` was set to be `LOMEM+&1800`; I have use way more memory than necessary due to long variable names. The code should be legible also neat'n'tidy with `LOCAL` variables - but that comes with a memory overhead. However, this just means more files need multiple passes; given its an automated disc-to-discs process, this has little impact.
