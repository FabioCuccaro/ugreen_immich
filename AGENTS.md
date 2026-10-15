# Rules for AI assistants and contributors

This repo is a public, generic guide: [Immich](https://immich.app) in Docker on a UGREEN NAS managed from the UGOS web UI, reading the photos the UGREEN app already syncs, reachable from outside only through Tailscale.

1. **Keep it generic.** No personal values: real IP addresses, Tailscale `100.x.y.z` addresses, device names, user names, personal paths, photo counts. Use placeholders between `< >` (`<NAS LAN IP>`, `<user>`, `<database password>`) or clearly fake examples (`192.168.1.10`).
2. **Never commit secrets**: database passwords, Immich API keys, filled-in compose files, the `library/` and `postgres/` folders.
3. **No internet exposure.** No UGOS Quick access or UGREENlink, no open router ports, no Tailscale Funnel. From outside, only through the tailnet (see [ugreen_tailscale](https://github.com/FabioCuccaro/ugreen_tailscale)).
4. **Check the official docs** (docs.immich.app, the immich-go repo, UGREEN) before writing variables, options, image tags or menu names. Immich has no `:stable` tag: pin the major version (`v3`). Mark anything not verified on a real NAS as untested (⚠️ in the README).
5. **Readers manage the NAS from the UGOS web UI**, often without SSH and with little terminal experience: write UGOS, Immich and phone steps one by one.
6. **English README is the entry point**, the Italian guides live in `docs/it/`. When a step changes, update both languages in the same commit.
7. When someone reports a result on a new model or setup, update the "Tested on" table in the README.
