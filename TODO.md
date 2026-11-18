# TODO

## Next

1. Choose how to build tide on Windows (SPEC.md §3, §6).
2. Answer SPEC.md §6's other open questions: how to hide a workspace, the
   taskbar, elevated windows and the first target PC.
3. Sign releases through SignPath Foundation's free open-source program
   (SPEC.md §5.1, §5.5): apply, add the signing policy it asks for, and
   have the release workflow submit each release. Its terms need a
   maintainer to approve every release in SignPath, so CI can't finish
   signing alone; decide whether that's acceptable first (SPEC.md §6).
4. Check the spec's *(unverified)* claims on a Windows 11 machine.

## Later

- Publish through the Microsoft Store as MSIX: free for an individual
  developer, signed by Microsoft, no SmartScreen warning, and package
  identity for reading notifications (SPEC.md §5.1, §5.4).

## Decisions needing review

None yet.
