# Hard Links, Soft Links, and Mounting

## Learning Objectives

By the end of this section you should be able to:
- Explain what a hard link is, and why deleting one name for a file doesn't necessarily delete the file's data
- Explain what a soft (symbolic) link is, and how it differs fundamentally from a hard link
- Explain what mounting does, and why it lets multiple physical storage devices appear as one unified directory tree

## Prerequisites

- Topic 2 (The Directory Abstraction)

## Motivation

Topic 2 established that a directory maps names to underlying files. This topic covers a natural, useful consequence of that design (multiple names can map to the *same* underlying file) and the related, but structurally different, idea of a symbolic reference — plus mounting, which extends the directory abstraction to unify entirely separate physical storage devices into one seamless tree.

## Problem Statement

If a directory entry is just a (name, reference) pair pointing at some underlying file (Topic 2), could **two different directory entries** — possibly even in different directories — point at the **exact same** underlying file? And separately: how does a file path like `/mnt/usb-drive/photo.jpg` seamlessly refer to a file on a completely different, physically separate storage device, using the exact same path syntax as any other file?

## Concept

### Hard Links

> A **hard link** is an additional directory entry (name, reference) pointing at the **same** underlying file as an existing entry — not a copy, and not a separate file, but a second (or third, or more) name for the identical underlying data.

The underlying file itself keeps a **reference count** of how many directory entries currently point to it. Deleting one name (one hard link) simply decrements this count and removes that specific directory entry — the underlying file's actual data is only truly deleted once its reference count reaches **zero** (no directory entries anywhere point to it anymore).

```
   Before:  /docs/report.txt  ──┐
                                    ├──► [ same underlying file, refcount=2 ]
            /backup/report.txt ──┘

   Delete "/docs/report.txt":

            /backup/report.txt ────► [ same underlying file, refcount=1 ]
            (the actual DATA is still fully intact — only ONE of its
             two names was removed)
```

This directly explains a detail that surprises many beginners: deleting a file's name doesn't necessarily delete its data at all — it only does so once the *last* remaining hard link to it is removed.

### Soft (Symbolic) Links

> A **soft link** (symbolic link, or "symlink") is a special kind of file whose contents are simply the **path** to another file — it's an indirect reference *by name*, not a direct reference to the same underlying file's data the way a hard link is.

```
   /shortcut  is a symlink, whose CONTENTS are the text "/docs/report.txt"

   Following /shortcut:  read its contents ("/docs/report.txt")
                          → then resolve THAT path (Topic 2) normally
```

### The Key Structural Difference

| | Hard link | Soft (symbolic) link |
|---|---|---|
| **What it points to** | The same underlying file directly (shares its reference count) | A path (a name), resolved separately, to whatever file currently exists at that path |
| **What happens if the original name is deleted** | No effect — the hard link still refers to the same underlying file, which still has at least one remaining name | **Broken** — the soft link's stored path no longer resolves to anything, since the original entry (and possibly the file itself) is gone |
| **Can it cross to a different physical storage device/mount?** | Generally, no — hard links typically must stay within the same underlying file system/device | Yes — since it's just a stored path, it can point anywhere the path resolution process (Topic 2) can reach, including a different mounted device |

A soft link, unlike a hard link, can become a **dangling reference** — pointing at a path that no longer resolves to anything at all — precisely because it's an indirect reference by name, not a direct, reference-counted link to the underlying file itself.

### Mounting

> **Mounting** attaches an entirely separate physical storage device's own directory tree onto a specific point (a directory) within an already-existing directory tree, making the two appear as one single, seamless, unified hierarchy — even though they're physically completely separate storage devices underneath.

```
   Before mounting a USB drive:

   /                    (main disk's own directory tree)
   ├── home/
   └── mnt/
        └── usb/         (an EMPTY directory, just a mount point)

   After mounting the USB drive at /mnt/usb:

   /                    (main disk)
   ├── home/
   └── mnt/
        └── usb/         (now transparently shows the USB
                            drive's OWN entire directory tree,
                            as if it had always been there)
             ├── photo.jpg
             └── notes.txt
```

Once mounted, path resolution (Topic 2) walking through `/mnt/usb/photo.jpg` transparently crosses from the main disk's directory structure onto the USB drive's entirely separate physical storage, with no special syntax or awareness required from the program doing the resolving — directly extending Module 06, Topic 1's transparency goal to multi-device storage.

## Internal Working (Preview)

```
   HARD LINK:                              SOFT LINK:

   Name A ──┐                              Name B ──► (contents: "path to Name A")
             ├──► [ underlying file,                        │
   Name C ──┘       refcount=2 ]                              ▼
                                                     resolved SEPARATELY, by
   Deleting Name A: refcount → 1,                    reading the path and
   underlying file STILL EXISTS                       looking it up again
   (reachable via Name C)                             (Topic 2's process)

                                            Deleting Name A: Name B's stored
                                            path now resolves to NOTHING —
                                            a DANGLING soft link


   MOUNTING:

   Main disk's tree:  /  →  /mnt/usb/  (an ordinary, empty directory...
                                          ...until mounted)
   USB drive's OWN separate tree:  /  →  photo.jpg, notes.txt

   AFTER mount(usb_drive, "/mnt/usb"):
   /mnt/usb/photo.jpg  now transparently resolves ONTO the USB
   drive's physically separate storage
```

## Real-World Analogy

A **hard link** is like two different, equally legitimate name tags both attached to the exact same physical box in a warehouse — removing one name tag doesn't touch the box itself; it's still there, findable by its other name tag, and only gets thrown out once every single name tag referring to it has been removed. A **soft link** is instead like a sticky note that just says "the box you want is over in aisle 7, box #42" — if aisle 7's box #42 is later removed or relabeled entirely, the sticky note still says the same thing, but following it now leads nowhere (a dangling reference). **Mounting** is like a large shopping mall that adds a new wing, physically built and managed by an entirely separate construction company, but designed so that walking from the mall's main corridor into that new wing feels completely seamless — visitors have no way to tell, just from walking around, that the new wing is a structurally separate building bolted onto the original.

## Why These Features Are Necessary

Hard links let a single underlying file be legitimately reachable from multiple locations without wasteful duplication (unlike copying the data, which would waste storage and require keeping multiple copies in sync). Soft links provide a more flexible, path-based reference that can cross storage device boundaries and reference things that may not even exist yet at the time the link is created. Mounting solves an entirely different problem: presenting genuinely separate physical storage devices as one seamless, unified directory hierarchy, so that programs (and the path-resolution process, Topic 2) never need any special awareness of which physical device a given part of the tree actually lives on.

## Advantages

- **Hard links** avoid data duplication while allowing a file to be legitimately reachable by multiple names, with automatic cleanup only once truly unreferenced (via the reference count).
- **Soft links** offer flexible, cross-device references, and can even point to paths that don't currently exist (useful for certain configuration/deployment patterns).
- **Mounting** lets an arbitrary number of physically separate storage devices present as one single, seamless directory tree, directly extending the transparency goal (Module 06, Topic 1) to multi-device storage.

## Disadvantages / Risks

- **Soft links can dangle** — silently pointing at nothing, if the target is ever moved, renamed, or deleted, since there's no reference-counting relationship tying the two together.
- **Hard links typically cannot cross file system/device boundaries**, limiting their use compared to soft links in multi-device setups.
- **Mount points must be carefully managed** — accessing a directory that should be a mount point before the actual device is mounted there can reveal an unexpected, empty (or entirely different) underlying directory instead.

## Best Practices

- Use hard links specifically when you need multiple genuinely equivalent names for the exact same underlying data within one storage device, with automatic cleanup handled by reference counting.
- Use soft links when you need a more flexible, cross-device reference, understanding that they can dangle if their target changes — always account for the possibility of a broken soft link in robust code.
- When troubleshooting "a directory that should have contents appears empty," check whether it's an unmounted mount point rather than assuming data loss.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "Deleting a file's only visible name always deletes its actual data immediately." | If other hard links to the same underlying file still exist, the data remains fully intact, reachable via those other names — the data is only actually deleted once the reference count reaches zero. |
| "Hard links and soft links are just two names for the same feature." | A hard link directly shares the same underlying file and reference count; a soft link is an independent file whose contents are merely a path, resolved separately each time, and can become a dangling reference if that path stops being valid. |

## Interview Questions

1. **Q: What is a hard link, and why doesn't deleting one name necessarily delete the file's data?**
   A: A hard link is an additional directory entry pointing at the same underlying file, which maintains a reference count of all entries pointing to it. Deleting one entry only decrements the count and removes that specific name; the actual data persists until the count reaches zero.

2. **Q: What's the key structural difference between a hard link and a soft link?**
   A: A hard link directly shares the same underlying file and its reference count. A soft link is a separate file whose contents are just a path, resolved independently each time it's followed — it can become a dangling reference if the target path stops resolving, which a hard link cannot.

3. **Q: What does mounting do, and why does it matter for the transparency of the file system?**
   A: Mounting attaches a separate physical storage device's directory tree onto a specific point within an existing tree, making them appear as one seamless hierarchy. This means path resolution can transparently cross from one physical device to another with no special syntax or awareness required, extending the OS's general transparency goal to multi-device storage.

## Summary

- A hard link is an additional name for the same underlying file, sharing its reference count; the file's data persists until every hard link to it is removed.
- A soft link is an independent file storing a path, resolved separately each time it's followed, and capable of dangling if that path becomes invalid.
- Mounting attaches a separate physical storage device's directory tree onto an existing hierarchy, making multiple physical devices appear as one seamless, unified tree.
- This closes out the module's coverage of the file and directory abstractions — the module summary ties files, directories, links, and mounting together before Module 20 covers how a real file system actually implements all of this on physical disk.
