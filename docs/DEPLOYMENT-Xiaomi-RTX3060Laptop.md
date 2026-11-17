# Deployment record — Xiaomi laptop / RTX 3060 Laptop GPU 6 GB (10de:2520)

> Machine profile for AI assistants: if you are given this repository and asked
> about "this machine", this file describes the exact hardware, the working
> gpu-unlock deployment, and the verification data. Treat it as the source of
> truth for this specific laptop.

## Machine identity (fingerprint)

| Field | Value |
| --- | --- |
| Model | Xiaomi laptop (Redmi line; `redmi_wmi` / `bitland_mifs_wmi` in kernel log) |
| CPU | AMD Ryzen 7 5800H |
| GPU | NVIDIA GeForce RTX 3060 Laptop GPU / Max-Q, **GA106M**, 6 GB |
| PCI IDs | `10de:2520` (VGA, `01:00.0`), audio `10de:228e` |
| PCI subsystem | `1d72:2037` |
| VBIOS | `94.06.1F.00.2D` |
| Display wiring | dGPU-wired: no iGPU function on PCI; internal display driven by the NVIDIA card |
| Chip architecture code | `(PMC_BOOT_0 >> 20) & 0x1ff = 0x176` (GA106 die) |

## Software stack (as deployed)

| Field | Value |
| --- | --- |
| OS | Arch Linux (rolling) |
| Kernel | `7.3.4-zen2-2-zen` (Zen kernel); previously CachyOS `7.3.0-3-cachyos` |
| NVIDIA driver | `nvidia-open-dkms 620.70.06` (open kernel module); previously proprietary `615.78.08` |
| Bootloader | **Limine 12.10.0** (not GRUB, not systemd-boot) |
| EFI partition | `/boot` (vfat), config at `/boot/EFI/BOOT/limine.conf` |
| root | btrfs subvolume `@`, `root=PARTUUID=<redacted>` |
| initramfs | mkinitcpio, image `/boot/initramfs-linux-zen.img` |
| Firmware source files | NVIDIA 610.43.03 `.tar.xz` + `.run`, downloaded to `~/下载/` |

## gpu-unlock deployment

- Version **2.0.0** (upstream HEAD `7eb51bc`).
- This fork's `main` carries one extra commit (`c6bb957`) that restores the
  GA106M device IDs dropped by the recipe-driven refactor (`09305c4`):
  `GA106(0x2520), GA106(0x2560)` in `kernel/gpu_unlock/recipes.c`.
  Without this patch the card is not recognized and no unlock happens.
- Built via `packaging/arch/PKGBUILD`:
  `cd packaging/arch && SRCDEST=/home/bruce/下载 makepkg -si`.
  `SRCDEST` lets makepkg reuse locally downloaded NVIDIA files (no network).
- Packages: `gpu-unlock-dkms 2.0.0-1` + `gpu-unlock-firmware 2.0.0-1`.
- `gpu_unlock.ko.zst` and `gpu-unlock/610.43.03/*` firmware confirmed inside
  `/boot/initramfs-linux-zen.img` via `lsinitcpio`.
- Kernel command line (in `limine.conf`, note the field is `cmdline:`):
  ```
  cmdline: root=PARTUUID=... zswap.enabled=0 rootflags=subvol=@ rw rootfstype=btrfs gpu_unlock.targets=all
  ```
- `/etc/modprobe.d/gpu-unlock.conf`:
  ```
  options nvidia NVreg_DynamicPowerManagement=0x00
  ```
  (keeps the dGPU from powering down at idle, which would reset the unlock).
- `mkinitcpio` drop-in `/etc/mkinitcpio.conf.d/zz-gpu-unlock.conf` loads
  `gpu_unlock` before `nvidia`. The old `gpu_unlock?` modules-load.d bug
  (upstream #29) is already fixed in current version.

## Power limit note

- Stock power limit is **95 W** (VBIOS default), hard max **130 W**.
- On `nvidia-open-dkms 620.70.06`, `nvidia-smi -pl 130` returns
  "Changing power management limit is not supported"; the open module does not
  expose the NVML power-limit control. 130 W was achieved by other means and a
  reboot preserved `enforced.power.limit = 130 W`.
- On the older proprietary `615.78.08` driver the 130 W cap was reachable in a
  different way (see issue #125).

## Verification evidence

### Kernel log (boot, successful unlock)

```
pci 0000:01:00.0: gpu_unlock: stage=claim device=0x2520 family=GA102/GA103/GA104/GA106 ...
pci 0000:01:00.0: gpu_unlock: stage=gfw plm=0xffffff8f ss0=0x72154070 ss1=0x6 ...
pci 0000:01:00.0: gpu_unlock: stage=host ss0=0x88888888 ss1=0x8 plm_feat=0xffffffff ...
pci 0000:01:00.0: gpu_unlock: stage=flr ... chip=0xb76000a1
pci 0000:01:00.0: gpu_unlock: unlocked
```
`ss0 0x72154070 -> 0x88888888` and `ss1 0x6 -> 0x8` confirm the speed
selectors were written full-rate and survived FLR.

### GEMM benchmark (8192^3 dense, FP32 accumulation, cuBLAS)

torch 2.12.1+cu130 in the ComfyUI venv:
`/home/bruce/ComfyUI-Installs/ComfyUI (1)/ComfyUI/.venv/bin/python`
Script kept at `/home/bruce/gpu_baseline.py` (home dir, not /tmp which gets wiped).

| Precision | Stock 95 W | Stock 130 W | Unlocked 130 W | Unlock gain |
| --- | --- | --- | --- | --- |
| FP16 | 24.91 | 27.93 | **45.20** | +62% |
| BF16 | 25.44 | 28.16 | **46.19** | +64% |
| TF32 | 12.48 | 13.67 | **23.25** | +70% |
| FP32 (control) | 8.05 | 9.26 | 9.22 | unchanged |

FP16/BF16/TF32 roughly double while FP32 is unchanged — the signature of a
correct tensor-core unlock. Result file: `/home/bruce/unlocked_result.txt`.

## Caveats / lessons learned

- **6 GB VRAM does not affect the unlock** (it changes speed selectors, not
  memory). But memory-bound workloads see no gain: LLM token generation
  (decode), INT8/FP8/INT4 models, gaming/FP32 all unchanged. Only compute-bound
  FP16/BF16/TF32 work (prefill, fine-tuning, image generation) benefits.
- ComfyUI Desktop holds a CUDA context and VRAM; close it before benchmarking.
- Reading `PMC_BOOT_0` from userspace (`/sys/bus/pci/.../resource0`) fails
  (`pread` I/O error, `mmap` EINVAL) because the NVIDIA driver owns BAR0. The
  chip value can be read from gpu-unlock's own boot log instead.
- To sync with a newer upstream later: `git remote add upstream https://github.com/Zagitta/gpu-unlock.git`,
  then `git fetch upstream && git rebase upstream/main`; the 3-line patch
  may need re-applying if upstream restructures `cards[]` again.
