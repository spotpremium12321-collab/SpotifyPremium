---
name: spotipw-binary-checks
description: Prove a Spotify class, selector, flag, metadata key or stored setting against the decrypted 9.1.90 app in ipa/ before a hook or a fix trusts it. Use when a claim depends on the shipped app rather than this repo's source, when a hook seems never to fire, or when a device report hinges on what Spotify actually gates.
---

# Proving it against the shipped app

`AGENTS.md` puts it as a rule: a class or selector is only real if a recorded tree or the binary says
so. This is how. The trees say how Spotify built a screen; the binary says what exists at all.

## One pass over the strings

    unzip -p ipa/com.spotify.client_9.1.90_und3fined.ipa "Payload/Spotify.app/Spotify" > /tmp/spotify-app
    strings -a /tmp/spotify-app | sort -u > /tmp/spotify-strings.txt

Then `grep -cE '^name$' /tmp/spotify-strings.txt` for a class, a selector, a defaults key, a flag or
a metadata key. One pass, then every question is fast; the unpacked binary is ~230 MB.

## What each name is

- `_TtC<module><len><Class>` is a Swift class (`_TtO` nested, `_TtP` protocol). The Kit cannot
  instance-subclass those (`subclassable()` in `Redesigned/Kit/SGRRestyle.m` refuses `_Tt` names), which
  is why hooks on Swift views hide by alpha or override the class instead.
- `scope.property` pairs are Spotify's remote config. The scope says who reads it:
  `ios-feature-settings.*` mostly draws settings rows; `core-*`, `playback-settings.*` and
  `core-connect-feature.*` are read by the core. The same property can exist in two scopes with
  different meaning — force both, or disassemble the caller (the AutoMix crossfade lesson:
  `docs/AUTOMIX-2026-11-10.md`).
- A cluster of names beside Swift type names is a struct or proto, not selectors:
  `enable_crossfade`, `crossfade_duration_ms` and `crossfade_time` are playback-settings fields,
  reachable only through the object that holds them.
- `automix.*`, `audio.fade_out_*`, `media.stop_position`, `canvas.*` are track metadata the backend
  sends: they are answers to a request, never guarantees.
- Defaults keys (`audio.crossfade_v2`) are what the preferences class reads. A settings screen may
  have migrated the value to another store, and a marker like `audio.crossfade_migrated_to_ps` means
  that copy happened once and does not follow later writes.
- Flags this app ships, with their values, are in `tweak/Sources/Shared/Flags/SGFlagList.m`
  (`make flags` regenerates it, about 40 s) — faster than grepping when the question is a default.

## When the names are not enough

`otool -tV` and a disassembler (Capstone from Python) for the caller of a config accessor or a
settings setter; a Swift property's offset comes from the type metadata, and reading one at runtime is
what `Redesigned/Kit/SGRDownload.m` does with `propertyBytes`. Say which of the three you used: strings
prove a name exists, disassembly proves who reads it, and neither proves what the phone hears.

## Limits

A name in the binary is not a hook that fires: prove the selector belongs to the class this repo
hooks, not merely to the app. Trees prove layout, the binary proves existence, the device proves
behaviour — a passing harness proves only what its fixtures feed it. When REA is available on the
machine (`~/.agents/skills/reverse-engineer-anything`), it is the heavy machinery for this work; it is
not vendored here because the copy carries no license.
