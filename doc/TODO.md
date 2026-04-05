# TODO

- Glossary abbreviations
- Usage - patterns
- Document why we didn't use mutagen/rsync/syncthing/git/juicefs/etc
- Auth - mutual trust model, no protection from bad actors, auth implies good actor
- How can git clones live inside the parlor while being worked on
- Plugin prompt/skill design - what do plugins expose to the coding agent, and what does the porter's prompt look like
- Porter roles and responsibilities - what it does, what it catches, what "using parlor wrong" looks like
- What AI agents of humans should know about parlor conventions
- Private dirs in patron dirs (likely simple AES encryption)
- Stub file format
- Can a stub represent a whole directory tree, not just a single file (overlaps with git support)
- Reconcile file sync timing with AI-generated change explanations
- Daemon-agent API/communication (conflicts, watch limit warnings, sync status, incoming changes, etc.)
- Daemon behavior when nearing inotify watch limits - notify agent
- Change stream: should a single entry be able to represent an entire dir (e.g. 1000 files added at once)
