> Is there any docs of how the fs work in source engine ?

Not really, from what I remember - though it's somewhat simple
(time for an info dump xd)

So, a search path is a specific path that contains the following information:
- m_storeId | A unique ID for this search path
- m_pPathIDInfo | information for the path (m_PathID & m_pDebugPathID - idk why those were separated into yet another layer)
- _flag0 | unknown
- m_GroupID | GMod Specific group ID of what this path is - like GN_ADDONCONTENT, GN_ENGINECORE or such. See CPathGroupName_t
- m_Path | A symbol for the g_PathIDTable for quick comparing and lookups
- m_pDebugPath | A string / the pathID Name but unreliable as in the case that it was never initialized it will contain an invalid pointer instead of being nullptr
- m_pPackFile | a pointer to the pack file which can be a .bsp, .zip or .vpk file (yes .zip is supported yet not actually used by any games iirc - also only partially implemented??? idk)
- m_pPackFile2 | I have absolutely no idea - this just seems to contain general data for a pack file?

Adding (CBaseFileSystem::AddSearchPathInternal):
A search path is registered by specifying the absolute path on disk to a folder or .vpk/.bsp

Searching (CBaseFileSystem::OpenForRead):

First - it's checked if the given path is an absolute file path on disk.
If so - it skips search paths and goes straight to reading / opening the file. If the path contains a .zip it will even read it - example valid path: C:\magicpath\GarrysMod\garrysmod\example.zip\sound\example.wav

If it isn't a absolute path, it proceeds to creating an iterator - and then checking each search path.
To filter search paths - it compares the CSearchPath::m_pPathIDInfo::m_PathID against the given pathID
(which previously was translated into a CUtlSymbol inside the CSearchPathsIterator constructor)

To search for a file - it goes through each search path and attempts to open it by calling CBaseFileSystem::FindFileInSearchPath

CBaseFileSystem::FindFileInSearchPath checks if the path is a pack file (.zip/.bsp/.vpk) and handle that - if it's a normal file, it proceeds to CBaseFileSystem::HandleOpenRegularFile
CBaseFileSystem::HandleOpenRegularFile is just an intermediate step for logging & such and proceeds to CBaseFileSystem::Trace_FOpen
CBaseFileSystem::Trace_FOpen is yet another intermediate step, and it proceeds to call CFileSystem_Stdio::FS_fopen

CFileSystem_Stdio::FS_fopen
on Windows only will first attempt to open a file (only if the mode is "rb") using the Windows API "CreateFile". This logic is inside the CWin32ReadOnlyFile class
if that fails, it falls back and attempts to open the file using fopen. This logic is inside the CStdioFile class

If after iterating all search paths it failed to open the file - it finished.
If it finds the file - it'll allocate a new CFileHandle and cast it to FileHandle_t and return it.