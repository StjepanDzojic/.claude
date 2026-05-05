---
name: iOS font path fix
description: GeneralSans fonts in the Xcode project were pointing to a developer's Downloads folder; fixed to use repo-relative path
type: project
---

Font references in `ios/beacon.xcodeproj/project.pbxproj` were originally pointing to `../../Downloads/GeneralSans_Complete/Fonts/OTF/` (a machine-specific Downloads folder).

**Why:** Fonts were added by dragging from the Downloads folder in Xcode instead of from within the repo.

**Fix applied:** Changed all 12 `PBXFileReference` paths to `../app/assets/fonts/GeneralSans-*.otf` with `sourceTree = "<group>"`. This resolves correctly because the root group resolves relative to the directory containing the `.xcodeproj` file (`ios/`), so `../app/assets/fonts/` points to `beacon/app/assets/fonts/` — the same for every developer.

**How to apply:** If fonts ever go red in Xcode again, check the `path` values for font `PBXFileReference` entries in `project.pbxproj` — they must use `../app/assets/fonts/` not an absolute or Downloads-relative path.
