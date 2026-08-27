# Notification Platform Technical Design Documents

Both deployables' designs live in this one directory, and the document identifier is what
states which system owns each: `TDD-notif-runtime-*` belongs to SAD-005, and
`TDD-notif-experience-*` belongs to SAD-015.

A subdirectory per system was tried and removed. The governance crawler's allowed-root set is
matched relative to the repository root, so designs nested under `apps/<system>/docs` are pruned
before any rule evaluates them — the documents lint as compliant without being read. One level
further down, `docs/designs/<system>/` is scanned and then fails `max_directory_depth`. The
identifier already carries ownership, so the directory was adding a level of nesting and no
information.