# HARDWARE DE LA VM — call_me_maybe

> Verificado 27/09, refinado 05/10. Este archivo vive en `docs/` (local, fuera
> del repo principal). Leer antes de medir latencia o cambiar config de la VM.

## Host y VM

- **Host**: Intel i7-7700HQ → **4 núcleos físicos / 8 hilos**, 24 GB RAM,
  GPU dedicada 4 GB (**NO** utilizable desde VirtualBox).
- **VM**: Ubuntu 22.04.5 (innotek 1.2), hostname `laviles-VirtualBox`,
  LVM 27 GB, RAM 15 GiB asignados.

## CPU — oversubscription y cap

- ⚠ La VM **tuvo** 6 vCPU sobre 4 núcleos físicos = oversubscription 1.5x.
  VirtualBox no expone hyperthreading al guest: ve 6 "núcleos" que en el
  host son 3 parejas SMT → contención de ALU/FPU/L1.
- 🔧 **Corregir (VM apagada)**: `VBoxManage modifyvm "laviles-VirtualBox" --cpus 4`
- 🔧 **Cap (en caliente)**: `VBoxManage modifyvm "laviles-VirtualBox" --cpu-executioncap 80`
  → deja 20% al host (Windows Defender + Update compiten por los 4 núcleos).
  **Per vCPU**. Se aplica con `controlvm`, NO con `modifyvm` (que exige VM apagada).
- **En código**: `threads=4` (1 thread por vCPU = 1 por núcleo físico).
- **Falsado 05/10**: la hipótesis "6 vCPU era la causa de la latencia" no se
  sostiene — la referencia limpia del 27/09 con 4 vCPU + cap 80 dio 292,72 s,
  mejor que 389,1 s con 6 vCPU. La regresión del 05/10 (325-614 s) es de
  entorno (energía/host), no de config de vCPU.

## GPU — no hay

- `lspci` → "VMware SVGA II Adapter" (emulada); `lsmod` sin nvidia/amdgpu/i915/nouveau.
- `torch 2.13.0+cpu`, `cuda.is_available()=False`, `torch.version.cuda=None`.
- Passthrough de VirtualBox es experimental (VT-d + IOMMU) → **no usar para ML**.
- El decoder es **device-agnostic**: `llm_sdk` resuelve device (mps > cuda > cpu)
  y dtype solo; `src/decoder/` es CPU puro. En un equipo con GPU corre sin tocar
  `src/`: `pip install torch accelerate` + `device_map="auto"`.
- **Formato de salida invariante al device**: header y tail se inyectan vía
  `model.encode()` sin `forward()`; la estructura la imponen `compute_allowed_ids`
  + `SchemaContext`; argmax sobre logits crudos (monótono, sin softmax).
- **Test de humo GPU** (pendiente, P4): guardar golden de CPU → correr en GPU →
  diffear. Si idéntico, cerrado.
- **Costo oculto en GPU**: `[float(x) for x in out.logits[0,-1].tolist()]` es
  no-op semántico que cuesta 26,4 ms/step sobre 151.643 elems (1% CPU, 17-53% GPU).

## RAM

- Host 24 GB − VM 15 GiB. **NO era 64 GB** (anotación vieja falsa).
- Peak medido de la suite: **4,65 GiB** (índice de vocabulario en RAM).
- ⚠ Monitores: `opencode` tiene `VmSize` 76 GB pero `VmRSS` real 800 MB.
  Usar VmRSS, nunca VmSize.

## Cómo medir limpio

1. `opencode` **cerrado** (contamina 2,6x — §1.1 de ESTADO_ACTUAL).
2. VM 4 vCPU, cap 80 para uso normal / cap 100% para medir techo.
3. Threads fijos en código: `threads=4`.
4. Referencia limpia de latencia (27/09): **292,72 s** · 3,364 cores efectivos ·
   7,40 CPU-s/forward · 133 forwards.
5. El KPI <5' se valida en **campus**, no local (decisión 05/10).
