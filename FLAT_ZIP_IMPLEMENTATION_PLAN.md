# Flat ZIP Preset Implementation Plan

## Overview
Add a "Flat ZIP" preset to the 7-Zip context menu that creates non-compressed ZIP archives with automatic settings.

## Feature Requirements
When user selects "Add Flat ZIP" from the context menu:
1. Format: ZIP
2. Compression: Store (0 - no compression)
3. Archive Name: FolderName.zip (auto-generated from selected folder)
4. Behavior: Exactly like manual process of:
   - Open folder
   - Ctrl + A (select all)
   - Right click → 7-Zip → Add to archive
   - Set Format: ZIP, Compression: Store

---

## Architecture Analysis

### Key Files & Components

#### 1. **ContextMenu.cpp** (Main Implementation)
- **Location**: `CPP/7zip/UI/Explorer/ContextMenu.cpp`
- **Key Functions**:
  - `QueryContextMenu()` (Line 585) - Builds menu items
  - `InvokeCommand()` (Line 1193) - Executes selected command
  - `InvokeCommandCommon()` (Line 1256) - Common execution logic for all commands

#### 2. **Command System Structure**
```cpp
struct CContextMenuCommand
{
  UInt32 flag;
  CZipContextMenu::enum_CommandInternalID CommandInternalID;
  LPCSTR Verb;
  UINT ResourceID;
};
```

- **Flags**: Defined in `ContextMenuFlags.h` (e.g., `kCompress`, `kCompressToZip`)
- **CommandInternalID**: Enum defining the command type
- **Verb**: String identifier for the command (e.g., "CompressToZip")
- **ResourceID**: Language string resource ID

#### 3. **Command Map Structure**
```cpp
struct CCommandMapItem
{
  enum_CommandInternalID CommandInternalID;
  UString Verb;
  UString Folder;
  UString ArcName;
  UString ArcType;
  bool IsPopup;
  UString UserString;
  // ... additional fields
};
```

#### 4. **Existing Compress Commands** (Reference)
From `g_Commands[]` array:
- `kCompressToZip` (Line 282) - Uses `CompressToZip` verb
- `kCompressToZipEmail` (Line 283) - Email variant
- Both use `IDS_CONTEXT_COMPRESS_TO` for menu text

### How Compress Commands Work

#### A. Query Phase (Lines 973-989)
```cpp
if (contextMenuFlags & NContextMenuFlags::kCompressToZip &&
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
  AddCommand(kCompressToZip, s, cmi);
  MyFormatNew_ReducedName(s, arcName_zip_Show);
  Set_UserString_in_LastCommand(s);
  MyInsertMenu(popupMenu, subIndex++, currentCommandID++, s, bitmap);
}
```

#### B. Invoke Phase (Lines 1297-1339)
```cpp
case kCompressToZip:
{
  UString arcName = cmi.ArcName;
  if (_fileNames_WereReduced)
  {
    UString arcName_base;
    arcName = CreateArchiveName(_fileNames, false, NULL, arcName_base);
    arcName += ".zip";
  }

  const bool email = cmdID == kCompressToZipEmail;
  const bool showDialog = cmdID == kCompress || cmdID == kCompressEmail;
  const bool addExtension = showDialog;
  CompressFiles(cmi.Folder,
      arcName, cmi.ArcType,
      addExtension,
      _fileNames, email, showDialog,
      false // waitFinish
  );
  break;
}
```

### Key Function: CompressFiles()
- **Location**: `CPP/7zip/UI/Common/CompressCall.h/cpp`
- **Parameters**:
  - `folder`: Output folder path
  - `arcName`: Archive filename (without extension if addExtension=true)
  - `arcType`: Format ("7z", "zip", etc.)
  - `addExtension`: Whether to auto-add extension
  - `fileNames`: Files/folders to compress
  - `email`: Send via email
  - `showDialog`: Show compression dialog
  - `waitFinish`: Wait for completion

---

## Implementation Steps

### Step 1: Add New Command ID & Flag
**File**: `ContextMenuFlags.h`
```cpp
const UInt32 kCompressToFlatZip = 0x00002000;  // New flag
```

**File**: `ContextMenu.h`
```cpp
enum enum_CommandInternalID
{
  // ... existing commands
  kCompressToFlatZip,  // Add new command ID
  // ...
};
```

### Step 2: Add Resource String
**File**: Resource files (`.rc` files in `CPP/7zip/UI/FileManager/`)
```
IDS_CONTEXT_COMPRESS_TO_FLAT_ZIP "Add to Flat ZIP"
```

### Step 3: Register Command in g_Commands Array
**File**: `ContextMenu.cpp` (around line 271)
```cpp
static const CContextMenuCommand g_Commands[] =
{
  // ... existing commands ...
  CMD_REC( kCompressToFlatZip, "CompressToFlatZip", IDS_CONTEXT_COMPRESS_TO_FLAT_ZIP),
};
```

### Step 4: Add Query Logic
**File**: `ContextMenu.cpp` (in `QueryContextMenu()` method)
- Add after the `kCompressToZip` block (after line 989)
- Check flag: `contextMenuFlags & NContextMenuFlags::kCompressToFlatZip`
- Create menu item with:
  - `ArcType = "zip"`
  - `ArcName = arcName_zip` (same naming)
  - Store compression level requirement

### Step 5: Add Invoke Logic
**File**: `ContextMenu.cpp` (in `InvokeCommandCommon()` method)
- Add case for `kCompressToFlatZip` in switch statement
- Call `CompressFiles()` with:
  - `arcName`: Archive name with .zip extension
  - `arcType`: "zip"
  - `showDialog`: false (no dialog, use preset)
  - But need to handle compression level preset

### Step 6: Modify CompressFiles() or Create Wrapper
**File**: `CPP/7zip/UI/Common/CompressCall.h/cpp`

**Option A**: Add compression preset parameter
```cpp
void CompressFiles(
    const UString &folder,
    const UString &arcName,
    const UString &arcType,
    bool addExtension,
    const UStringVector &fileNames,
    bool email,
    bool showDialog,
    bool waitFinish,
    const UString &compressionMethod = L""  // NEW parameter
);
```

**Option B**: Create dedicated function
```cpp
void CompressFilesAsStore(
    const UString &folder,
    const UString &arcName,
    const UStringVector &fileNames
);
```

### Step 7: Handle Compression Settings
The compression settings (Store/0) need to be passed to the compression engine via:
- Registry-based settings that are auto-configured before compression
- Or command-line parameters to the compression executable
- Or modification of the dialog/preset mechanism

---

## Key Considerations

### 1. Menu Visibility
- Add registry flag check in `CContextMenuInfo` to enable/disable
- Default: enabled
- User can control via Options dialog

### 2. File Selection Handling
- Works with single or multiple files/folders
- Uses same archive naming as `kCompressToZip`
- Auto-generates folder name from selection

### 3. Compression Level Persistence
- Store compression method as registry setting
- Or create temporary preset that applies Store compression
- Ensure dialog doesn't appear (showDialog=false)

### 4. Error Handling
- Validate folder exists
- Handle long filenames
- Check disk space

### 5. Localization
- Add resource strings for all languages
- Keep consistent with existing "Add to archive" naming

---

## Testing Checklist

- [ ] Menu item appears when right-clicking folder
- [ ] Menu item appears when right-clicking files
- [ ] Menu item appears when right-clicking multiple items
- [ ] ZIP created with Store (0) compression
- [ ] Archive filename matches folder/selection name
- [ ] No compression dialog appears
- [ ] Works from Explorer context menu
- [ ] Works from 7-Zip File Manager
- [ ] Works with long filenames
- [ ] Works with special characters in names
- [ ] Cascaded menu configuration respected
- [ ] Icon appears in menu (if enabled)

---

## Related Files to Modify

1. `CPP/7zip/UI/Explorer/ContextMenu.h` - Add enum value
2. `CPP/7zip/UI/Explorer/ContextMenu.cpp` - Main implementation
3. `CPP/7zip/UI/Explorer/ContextMenuFlags.h` - Add flag
4. `CPP/7zip/UI/FileManager/MenuPage2.rc` - UI configuration
5. `CPP/7zip/UI/FileManager/MenuPageRes.h` - Resource IDs
6. `CPP/7zip/UI/Common/CompressCall.h/cpp` - Compression function
7. Language resource files (`.rc` files)
