# Woow immich — MOVED / 已遷移

> [!IMPORTANT]
> **This repository has been split into per-platform repositories and is now
> archived.** The branch-per-platform layout is retired; each deployment now
> lives in its own repo:
>
> | Platform | New repository | Replaces branch |
> |----------|----------------|-----------------|
> | Docker / Podman Compose | [**Woow_podman_immich**](https://github.com/WOOWTECH/Woow_podman_immich) | `podman` |
> | K3s / Kubernetes (now a **Helm chart**) | [**Woow_k3s_immich**](https://github.com/WOOWTECH/Woow_k3s_immich) | `k3s` |
> | Home Assistant add-on | [**Woow_ha_immich**](https://github.com/WOOWTECH/Woow_ha_immich) | `ha` |
>
> **Home Assistant users:** remove this repository's URL from your add-on
> repositories and add the new one instead — archived repos never receive updates.
>
> **Note:** `Woow_k3s_immich` has a known upstream drift issue with pgvecto-rs
> vs current immich-server ([issue #1](https://github.com/WOOWTECH/Woow_k3s_immich/issues/1))
> pending an image tag bump.
>
> 本倉庫已依部署平台拆分為獨立倉庫並封存，原分支內容（含完整 git 歷史）已遷移至
> 上表新倉庫；K3s 版本並已改為 Helm chart。HA add-on 使用者請移除本倉庫網址，
> 改加入新倉庫，否則不會再收到更新。

---

The original branches remain readable here for reference, but all future
development happens in the new repositories.
