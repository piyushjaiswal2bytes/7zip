# Detailed Code Analysis: Flat ZIP Preset Feature

## Files Examined Summary

### 1. ContextMenuFlags.h
**Location**: `CPP/7zip/UI/Explorer/ContextMenuFlags.h`  
**Purpose**: Defines flags for enabling/disabling context menu commands

```cpp
namespace NContextMenuFlags
{
  const UInt32 kExtract = 1 << 0;
  const UInt32 kExtractHere = 1 << 1;
  const UInt32 kExtractTo = 1 << 2;
  const UInt32 kTest = 1 << 4;
  const UInt32 kOpen = 1 << 5;
  const UInt32 kOpenAs = 1 << 6;
  
  // Compression flags
  const UInt32 kCompress = 1 << 8;
  const UInt32 kCompressTo7z = 1 << 9;
  const UInt32 kCompressEmail = 1 << 10;
  const UInt32 kCompressTo7zEmail = 1 << 11;
  const UInt32 kCompressToZip = 1 << 12;
  const UInt32 kCompressToZipEmail = 1 << 13;
  
  const UInt32 kCRC_Cascaded = (UInt32)1 << 30;
  const UInt32 kCRC = (UInt32)1 << 31;
}
```

**Key Insight**: Flags use bit-shifting pattern. Next available flag for kCompressToFlatZip: `1 << 14`

---

### 2. ContextMenu.h
**Location**: `CPP/7zip/UI/Explorer/ContextMenu.h`

#### Command ID Enum (Lines 74-101)
```cpp
enum enum_CommandInternalID
{
  kCommandNULL,
  kOpen,
  kExtract,
  kExtractHere,
  kExtractTo,
  kTest,
  kCompress,
  kCompressEmail,
  kCompressTo7z,
  kCompressTo7zEmail,
  kCompressToZip,
  kCompressToZipEmail,
  kHash_CRC32,
  kHash_CRC64,
  // ... hash commands ...
  kHash_TestArc
};
```

**Key Insight**: Must add `kCompressToFlatZip` enum value

#### CCommandMapItem Structure (Lines 115-138)
```cpp
struct CCommandMapItem
{
  enum_CommandInternalID CommandInternalID;
  UString Verb;
  UString UserString;
  UString Folder;
  UString ArcName;
  UString ArcType;
  bool IsPopup;
  enum_CtxCommandType CtxCommandType;
};
```

**Key Insight**: 
- `ArcType`: Stores format ("zip", "7z", etc.)
- `ArcName`: Archive filename
- `Folder`: Output folder path
- `UserString`: Display text in menu

---

### 3. CompressCall.h
**Location**: `CPP/7zip/UI/Common/CompressCall.h`

```cpp
HRESULT CompressFiles(
    const UString &arcPathPrefix,      // Output folder
    const UString &arcName,            // Archive name
    const UString &arcType,            // Format ("zip", "7z")
    bool addExtension,                 // Auto-add extension
    const UStringVector &names,        // Files to compress
    bool email,                        // Send via email
    bool showDialog,                   // Show dialog
    bool waitFinish);                  // Wait for completion
```

**Key Insight**: This is the main function that executes compression. Parameters are passed as command-line switches to 7zG.exe.

---

### 4. CompressCall.cpp
**Location**: `CPP/7zip/UI/Common/CompressCall.cpp`

#### CompressFiles Implementation (Lines 186-240)

```cpp
HRESULT CompressFiles(
    const UString &arcPathPrefix,
    const UString &arcName,
    const UString &arcType,
    bool addExtension,
    const UStringVector &names,
    bool email, bool showDialog, bool waitFinish)
{
  MY_TRY_BEGIN
  UString params ('a');  // 'a' = add command
  
  // Create file mapping for file list
  CFileMapping fileMapping;
  NSynchronization::CManualResetEvent event;
  params += kIncludeSwitch;
  RINOK(CreateMap(names, fileMapping, event, params))

  // Add archive type
  if (!arcType.IsEmpty())
  {
    params += kArchiveTypeSwitch;  // " -t"
    params += arcType;
  }

  // Email mode
  if (email)
    params += kEmailSwitch;  // " -seml."

  // Show dialog
  if (showDialog)
    params += kShowDialogSwitch;  // " -ad"

  AddLagePagesSwitch(params);

  // Extension handling
  if (arcName.IsEmpty())
    params += " -an";  // No archive name
  else
  {
    if (addExtension)
      params += " -saa";  // Auto-add archive extension
    else
      params += " -sae";  // No extension
  }

  params += kStopSwitchParsing;  // " --"
  params.Add_Space();
  
  // Add archive path
  if (!arcName.IsEmpty())
  {
    params += GetQuotedString(arcPathPrefix + arcName);
  }
  
  // Call 7zG.exe with parameters
  return Call7zGui(params, waitFinish, &event);
  MY_TRY_FINISH
}
```

**Key Insights**:
1. **Command-line approach**: Passes parameters to 7zG.exe
2. **No compression level control**: Current function doesn't support compression method presets
3. **Parameters passed**:
   - `a` = add/compress command
   - `-t{type}` = archive type (zip, 7z, etc.)
   - `-ad` = show dialog
   - `-seml.` = email mode
   - `-saa`/`-sae` = extension handling

---

## Compression Settings Problem & Solution

### Problem
The `CompressFiles()` function doesn't currently support compression method presets like "Store (0)". The dialog is shown if `showDialog=true`, or it uses previously saved settings.

### Options to Add Store Compression

#### Option 1: Add Compression Method Parameter (RECOMMENDED)
**Modify CompressCall.h signature:**
```cpp
HRESULT CompressFiles(
    const UString &arcPathPrefix,
    const UString &arcName,
    const UString &arcType,
    bool addExtension,
    const UStringVector &names,
    bool email,
    bool showDialog,
    bool waitFinish,
    const UString &compressionMethod = L"");  // NEW
```

**In CompressCall.cpp:**
```cpp
if (!compressionMethod.IsEmpty())
{
  params += " -m0=";
  params += compressionMethod;  // e.g., "Copy" for Store/0
}
```

#### Option 2: Use Registry-Based Preset
Store compression settings in registry and read before calling CompressFiles.

#### Option 3: Create Dedicated Function
```cpp
HRESULT CompressFilesAsStore(
    const UString &arcPathPrefix,
    const UString &arcName,
    const UStringVector &names);
```

**Recommended Approach**: Option 1 (backward compatible, flexible)

---

## 7-Zip Command-Line Parameters

### ZIP Compression Methods
From `Methods.txt` documentation (lines 84-99):

```
04..01 - [Zip]
  00 - Copy (Store - 0 compression)
  01 - Shrink
  08 - Deflate
  09 - Deflate64
  0E - LZMA (LZMA-zip)
  5D - ZSTD
```

### 7zG.exe Command Examples
```bash
# Standard ZIP with dialog
7zG.exe a -t zip -ad archive.zip files...

# ZIP with Store (no compression) - no dialog
7zG.exe a -t zip -m0=Copy -sae archive.zip files...

# ZIP with Store compression level
7zG.exe a -t zip -m0=Copy archive.zip files...
```

### Parameter Reference
- `-t {type}` = Archive type (zip, 7z, etc.)
- `-m0=Copy` = Compression method (Copy = Store/no compression)
- `-m0d=...` = Dictionary size
- `-m1=...` = Second method (for filters)
- `-ad` = Show compression dialog
- `-sae` = No extension auto-add
- `-an` = No archive

---

## Implementation Approach - Detailed Steps

### Step 1: Add Flag to ContextMenuFlags.h
```cpp
const UInt32 kCompressToFlatZip = 1 << 14;
```

### Step 2: Add Command ID to ContextMenu.h
In enum_CommandInternalID, add after `kCompressToZipEmail`:
```cpp
kCompressToFlatZip,
```

### Step 3: Register Command in g_Commands Array (ContextMenu.cpp)
Around line 283, add:
```cpp
CMD_REC( kCompressToFlatZip, "CompressToFlatZip", IDS_CONTEXT_COMPRESS_TO_FLAT_ZIP),
```

### Step 4: Add Menu Item Creation Logic (ContextMenu.cpp::QueryContextMenu)
After line 1004 (after CompressToZipEmail block), add:

```cpp
// CompressToFlatZip
if (contextMenuFlags & NContextMenuFlags::kCompressToFlatZip &&
    !arcName_zip.IsEqualTo_NoCase(fs2us(fi0.Name)))
{
  CCommandMapItem cmi;
  UString s;
  if (_dropMode)
    cmi.Folder = _dropPath;
  else
    cmi.Folder = fs2us(folderPrefix);
  cmi.ArcName = arcName_zip;
  cmi.ArcType = "zip";
  AddCommand(kCompressToFlatZip, s, cmi);
  MyFormatNew_ReducedName(s, arcName_zip_Show);
  Set_UserString_in_LastCommand(s);
  MyInsertMenu(popupMenu, subIndex++, currentCommandID++, s, bitmap);
}
```

### Step 5: Add Command Execution Logic (ContextMenu.cpp::InvokeCommandCommon)
In the switch statement (around line 1302), add:

```cpp
case kCompressToFlatZip:
{
  UString arcName = cmi.ArcName;
  if (_fileNames_WereReduced)
  {
    UString arcName_base;
    arcName = CreateArchiveName(_fileNames, false, NULL, arcName_base);
    arcName += ".zip";
  }

  const bool showDialog = false;  // No dialog for preset
  const bool addExtension = false;
  CompressFiles(cmi.Folder,
      arcName, cmi.ArcType,
      addExtension,
      _fileNames, 
      false,  // email
      showDialog,
      false   // waitFinish
  );
  break;
}
```

### Step 6: Add Compression Method Parameter

**Modify CompressCall.h:**
```cpp
HRESULT CompressFiles(
    const UString &arcPathPrefix,
    const UString &arcName,
    const UString &arcType,
    bool addExtension,
    const UStringVector &names,
    bool email, 
    bool showDialog, 
    bool waitFinish,
    const UString &compressionMethod = L"");  // NEW
```

**Modify CompressCall.cpp (lines 186-240):**
```cpp
HRESULT CompressFiles(
    const UString &arcPathPrefix,
    const UString &arcName,
    const UString &arcType,
    bool addExtension,
    const UStringVector &names,
    bool email, 
    bool showDialog, 
    bool waitFinish,
    const UString &compressionMethod)  // NEW
{
  // ... existing code ...
  
  if (!arcType.IsEmpty())
  {
    params += kArchiveTypeSwitch;  // " -t"
    params += arcType;
  }

  // Add compression method if specified
  if (!compressionMethod.IsEmpty())
  {
    params += " -m0=";
    params += compressionMethod;
  }

  // ... rest of existing code ...
}
```

### Step 7: Update Invoke Call for FlatZip
Modify the case statement to pass compression method:

```cpp
case kCompressToFlatZip:
{
  UString arcName = cmi.ArcName;
  if (_fileNames_WereReduced)
  {
    UString arcName_base;
    arcName = CreateArchiveName(_fileNames, false, NULL, arcName_base);
    arcName += ".zip";
  }

  const bool showDialog = false;  // No dialog for preset
  const bool addExtension = false;
  CompressFiles(cmi.Folder,
      arcName, cmi.ArcType,
      addExtension,
      _fileNames, 
      false,      // email
      showDialog,
      false,      // waitFinish
      L"Copy"     // compressionMethod (Store/no compression)
  );
  break;
}
```

### Step 8: Add Resource String
In resource files (`.rc` files), add:
```
IDS_CONTEXT_COMPRESS_TO_FLAT_ZIP "Add to Flat ZIP"
```

### Step 9: Enable in Context Menu Settings
Modify `MenuPage2.rc` or settings code to allow enabling/disabling this option.

---

## Key Files Summary for Modification

| File | Lines | Changes |
|------|-------|---------|
| ContextMenuFlags.h | 21 | Add `kCompressToFlatZip = 1 << 14` |
| ContextMenu.h | 87 | Add `kCompressToFlatZip` to enum |
| ContextMenu.cpp | 283 | Add command to g_Commands array |
| ContextMenu.cpp | 1005 | Add menu item query logic |
| ContextMenu.cpp | 1340 | Add command invoke logic |
| CompressCall.h | 16 | Add compressionMethod parameter |
| CompressCall.cpp | 192 | Add compressionMethod parameter & logic |
| Resource files | Various | Add IDS_CONTEXT_COMPRESS_TO_FLAT_ZIP string |

---

## Flow Diagram

```
User Right-Clicks → QueryContextMenu() Called
  ↓
Check flag: NContextMenuFlags::kCompressToFlatZip
  ↓
Create CCommandMapItem:
  - CommandInternalID = kCompressToFlatZip
  - ArcType = "zip"
  - ArcName = "FolderName.zip"
  - Folder = output folder
  ↓
Add to Menu Display
  ↓
User Clicks "Add to Flat ZIP"
  ↓
InvokeCommand() Called
  ↓
InvokeCommandCommon() Called with CCommandMapItem
  ↓
case kCompressToFlatZip:
  ↓
CompressFiles(folder, "FolderName.zip", "zip", false, files, false, false, false, "Copy")
  ↓
Call7zGui() with params: "a -t zip -m0=Copy -sae -- folder/FolderName.zip"
  ↓
7zG.exe Executes Compression (Store method, no dialog)
```

---

## Testing Command Line

To test compression with Store method:
```bash
7zG.exe a -t zip -m0=Copy -sae -- C:\output\FolderName.zip C:\folder\*
```

Expected Result:
- ZIP file created with 0% compression (Store method)
- No dialog shown
- All files stored as-is

---

## Localization Notes

Add translations for:
```
IDS_CONTEXT_COMPRESS_TO_FLAT_ZIP = "Add to Flat ZIP"
```

Should be added to all language resource files in the repository.

---

## Related Documentation Files

- `DOC/Methods.txt` - Compression method IDs
- `DOC/7zC.txt` - 7z decoder API documentation
- `DOC/7zFormat.txt` - 7z format specification
