# Running
## Ragnarok online client
- Search for `iRO_ver12.0-full-20081229-1155.exe` client, install it to get the assets

## Mac OSX

Install RO client using Whisky
Set env variable `OPEN_MIDGARD_DATA_DIR` to point to the installtion path


Environment variables:
```
# Point this to RO installation path
export OPEN_MIDGARD_DATA_DIR="/Users/aharijanto/Library/Containers/com.isaacmarovitz.Whisky/Bottles/D7FF8420-1145-417B-8B4F-231ABE8F1BCE/drive_c/Program Files/Gravity/RagnarokOnline"

# Vulkan-related paths
export DYLD_FALLBACK_LIBRARY_PATH="/opt/homebrew/lib:$DYLD_FALLBACK_LIBRARY_PATH"
export VK_DRIVER_FILES="/opt/homebrew/share/vulkan/icd.d/MoltenVK_icd.json"
export VK_DRIVER_FILES=/opt/homebrew/etc/vulkan/icd.d/MoltenVK_icd.json
```

Running
```
./build-macos/open-midgard
```