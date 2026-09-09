# Per-map global Stripper filters

AnneHappy uses `cfg/stripper/zonemod_anne`. The complete source content lives at
`scripts/stripper/global_filters/zonemod_anne.cfg`. The generator keeps global
weapon, medicine, throwable, pickup, and competitive item policy in the runtime
`global_filters.cfg`; model, collision, environment, and map-entity sections are
prepended only to existing root Anne map configs. Other Stripper modes keep
their original global and map configs unchanged.

Official Valve campaigns (`c1`–`c14`) already have matching anne nav, so their
map configs may keep zonemod clip/blocker/ladder `add:`s. For custom maps, do
not add ladders, solid props, or nav-blocker entities unless the corresponding
anne nav (or a covering `nav_fixes` script) is updated in the same change.
`env_physics_blocker` and `env_player_blocker` may be kept if `BlockType` is
survivors only (`1`).

Stripper loads map configs at every underscore-delimited prefix. A map config
that already inherits from a shorter existing config must not contain another
generated global block. The generator detects that relationship automatically.

Regenerate and verify from the repository root:

```sh
python3 scripts/stripper/expand_global_filters.py
python3 scripts/stripper/expand_global_filters.py --check
```

When changing a shared filter, edit its file under
`scripts/stripper/global_filters/`, regenerate, and commit both the source and
expanded map configs.
