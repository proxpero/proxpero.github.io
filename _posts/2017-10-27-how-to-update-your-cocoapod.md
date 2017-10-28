---
title: "How to update your Cocoapod"
summary: "A checklist to help me (and possibly you) remember what to do to update a Cocoapod that you own."
tags: [Xcode, Git, Cocoapods]
toc:
---
 1. Run a `git pull` to refresh your local repository with the latest changes on the remote.
 2. Make changes and local commits.
 3. Bump the `s.version` in the `.podspec` file.
 4. Run `pod lib lint` to test that nothing broke for Cocoapods.
 5. Commit the final change to the `.podspec`
 6. Push the changes to Github.
 7. Bump the version in Github (be sure to match the new `.podspec` version).
 8. Run `pod trunk push {ProjectName}.podspec`.
