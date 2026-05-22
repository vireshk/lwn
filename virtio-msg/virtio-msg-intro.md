# Virtio goes message-based: introducing virtio-msg

A long-running effort to extend Virtio beyond PCI and MMIO register access is
converging on a formal specification and a growing Linux prototype. Bill Mills
[posted](https://lore.kernel.org/all/20260126163230.1122685-1-bill.mills@linaro.org/)
the first non-RFC Virtio specification patch series to
`virtio-comment@lists.linux.dev` on 26 January 2026. In parallel, a Linux
[implementation](https://git.kernel.org/pub/scm/linux/kernel/git/vireshk/linux.git/log/?h=virtio/msg)
(on branch `virtio/msg-amp` for AMP hardware support) implements the common
transport, several bus backends, and early asymmetric-multiprocessing paths.

The work sits under Linaro's **HVAC** (Heterogeneous VirtIO for Automotive
Computing) umbrella, with contributors from Linaro, Arm, AMD, ST, Google, and
others. The goal is not to replace virtio-pci or virtio-mmio where those work
well, but to give Virtio a transport that fits software-only peers,
heterogeneous SoCs, secure partitions, and multi-system links where
trap-and-emulate register access is awkward or impossible.

---

## When PCI and MMIO are the wrong abstraction

Virtio today is usually carried by **virtio-pci** or **virtio-mmio**. Both
assume the driver can perform transport operations through memory-mapped reads
and writes (or PCI config space): feature bits, status, config space, queue
addresses, and notifications map to well-known register layouts.

That model breaks down in several deployments the HVAC group cares about:

- **Host and co-processor** on an AMP SoC (application processor plus MCU, DSP,
  or firmware core), talking over shared memory and doorbells rather than a
  guest-visible virtio-pci device.
- **Two separate machines** linked by PCIe (not cache-coherent SMP), each
  running its own OS.
- **Normal and secure worlds** (Rich OS ↔ TEE) where the channel is an IPC ABI
  such as Arm FF-A, not MMIO the Rich OS can bit-bang freely.
- **Host ↔ VM or VM ↔ VM** where the hypervisor exposes FF-A or another message
  channel instead of emulated virtio-mmio.
- **Userspace-implemented devices** in the same address space as the kernel,
  where there is no hardware to trap on.

In all of these cases, driver and device are often **implemented in software on
both sides**, but there is no natural place to hang a virtio-mmio register block
or a virtio-pci BAR. What you do have is a **message channel**: hypercalls, FF-A
direct messages, shared-memory queues with interrupts, Xen events and grants, or
similar.

**virtio-msg** is the proposed answer: keep Virtio's device model (features,
config space, virtqueues, standard device types like net and blk), but implement
the **transport layer** by exchanging structured messages instead of MMIO-style
register access.

---

## Two layers: transport vs bus

The design deliberately splits responsibilities between transport and bus.

### Transport layer (common / device level)

This is what the Virtio specification patch series focuses on first. It defines
a fixed set of **transport messages** between a Virtio driver and a device
instance:

- Discovery and negotiation: `GET_DEVICE_INFO` / `GET_DEVICE_FEATURES` /
  `SET_DRIVER_FEATURES` / `GET_DEVICE_STATUS` / `SET_DEVICE_STATUS`.
- Configuration: `GET_CONFIG` / `SET_CONFIG`, with generation counts as in core
  Virtio.
- Virtqueues: `GET_VQUEUE` / `SET_VQUEUE` / `RESET_VQUEUE` (queue sizes and ring
  addresses, typically guest-physical or bus-defined).
- Events: `EVENT_CONFIG` / `EVENT_AVAIL` / `EVENT_USED` (and optional polling
  where events are not delivered).
- Shared memory IDs: `GET_SHM` aligned with core Virtio shared-memory regions.

Messages share an **8-byte header** (`type`, `msg_id`, `dev_id`, `token`, and
`msg_size`) plus payload. Request/response pairing uses `msg_id` and an optional
**token** when multiple outstanding requests are allowed. Revision 1 advertises
message sizes from 52 bytes up to 65535, with **264 bytes recommended** for
interoperability (enough for typical inline config in one round trip).

The transport preserves **normal Virtio semantics** after setup: existing
virtio-net, virtio-blk, and similar device drivers should not need
virtio-msg-specific changes—only the transport and bus plumbing change.

### Bus layer

The **bus** carries bytes between driver and device sides. It also owns platform
concerns: discovery, DMA mapping, hotplug, and how "physical" addresses in
virtqueue setup are interpreted.

The spec takes a **middle path** for buses:

- Optional standard bus messages: `GET_DEVICES`, `PING`, `EVENT_DEVICE` (hotplug
  ADDED/REMOVED).
- Room for **bus-specific** messages where environments differ (FF-A memory
  lend/share, Xen grants, PCIe vendor protocols).

Buses may discover devices via messages, device tree, ACPI, or hypervisor
tables; once a **device number** exists, transport messages are forwarded
opaquely. Because device numbers are 16-bit, a single bus instance supports on
the order of 65,000 devices; larger deployments use **multiple bus instances**
rather than wider IDs (as described in the spec).

#### Bus implementations

The Linux tree includes several bus drivers under `drivers/virtio/`.

The **FF-A bus** targets normal↔secure, host↔guest, and guest↔guest when the
hypervisor exposes FF-A. It integrates `memory_share` / lend-style sharing for
buffers, optional **restricted DMA pools** via device tree.

The **loopback bus** echoes messages for development—similar in spirit to
fuse, cuse, or loopback block devices, but for full virtio devices.

The **AMP bus** (on `virtio/msg-amp`) is the flavor for **shared RAM +
bidirectional notification**:

- Fixed **64-byte** messages (a cache-line-friendly control-plane size).
- **Lockless SPSC** rings in shared memory with a **self-describing layout**:
  each side owns queue heads, notification bitmaps, and element buffers.
- Generic **PCI** and **Sapphire** drivers map BARs, RAM, and MSI/legacy IRQs.

---

## Specification status

Bill's January 2026 posting is **[PATCH v1 0/4]** to virtio-comment—the first
formal upstream spec proposal after RFC rounds. The series adds `transport-msg.tex`
to the Virtio specification; a follow-on revision is expected on the same list.

Arm has also published a **Virtio Message Bus over FF-A** document separately
from the OASIS virtio-msg transport chapter—implementations already demo Trusty
and OP-TEE.

---

## Ecosystem and hardware demos

The [cover letter](https://lore.kernel.org/all/20260126163230.1122685-1-bill.mills@linaro.org/)
and [HVAC project materials](https://linaro.atlassian.net/wiki/spaces/HVAC/overview)
point to a deliberately broad prototype stack:

- **[hvac-demo](https://github.com/wmamills/hvac-demo)** — QEMU and userspace
  loopback examples.
- **[QEMU](https://github.com/edgarigl/qemu/commits/edgar/virtio-msg-rfc/)** — virtio-msg-amp device work (RFC on qemu-devel, 2025).
- **[open-amp](https://github.com/arnopo/open-amp/commits/virtio-msg/)** —
  RTOS-side AMP messaging.
- **Hardware**:
  - AMD x86 host + AMD Versal (Arm) over PCIe.
  - ST STM32MP157: Linux on Cortex-A7 with virtio-i2c provided by Cortex-M4
    (Zephyr).

Planned or discussed buses include **virtio-msg-xen** (grants + events,
including x86 Xen without FF-A) and **virtio-msg over admin virtqueues** on
virtio-pci (sub-devices on existing PCI virtio)—the latter still seeking
collaborators.

---

## What is done vs what remains

### Done (prototype quality)

- Full `transport-msg.tex` in virtio-spec (v1 posted to virtio-comment).
- Linux common virtio-msg transport, FF-A and loopback buses, AMP (shared memory
  and PCI/Sapphire platform drivers), and a userspace miscdevice interface.
- Demos: FF-A with pKVM/Xen and Trusty/OP-TEE; AMP on two hardware platforms;
  loopback and QEMU paths.

### In flight / open questions

- Spec review on virtio-comment.
- Linux RFC merge path (earlier LKML RFC from 2025; AMP branch is additive).
- Alignment of all demos with v1 spec (acknowledged in cover letter).
- Xen bus, admin-vq bus, standardized PCI-for-AMP bus text in OASIS spec.

---

## Why this matters for readers

For kernel developers, virtio-msg is the missing **transport type** when both
ends are software but the link is FF-A, shared memory, or a vendor PCIe
mailbox—not a virtio-mmio window in guest physical address space. The split
between **one reusable transport implementation** in the kernel and **many thin
bus drivers** is the main architectural win; AMP and FF-A are early demonstrations
on real hardware and TEEs.

For automotive and embedded SoC teams (HVAC's original audience), the same
mechanism covers **MCU ↔ AP** virtio devices (I2C, sensors, accelerators)
without bespoke IPC protocols for every device class—provided the industry
converges on virtio-msg buses for each physical link.

---

## References

| Item | Link |
|------|------|
| Virtio spec v1 cover (Jan 2026) | [lore.kernel.org — Bill Mills](https://lore.kernel.org/all/20260126163230.1122685-1-bill.mills@linaro.org/) |
| Linux `virtio/msg` | [git.kernel.org](https://git.kernel.org/pub/scm/linux/kernel/git/vireshk/linux.git/log/?h=virtio/msg) |
| Linux `virtio/msg-amp` | [git.kernel.org](https://git.kernel.org/pub/scm/linux/kernel/git/vireshk/linux.git/log/?h=virtio/msg-amp) |
| Linux RFC (2025) | [lore.kernel.org — Viresh Kumar](https://lore.kernel.org/all/cover.1753865268.git.viresh.kumar@linaro.org/) |
| HVAC overview | [Linaro Confluence — HVAC](https://linaro.atlassian.net/wiki/spaces/HVAC/overview) |
| Demos | [hvac-demo](https://github.com/wmamills/hvac-demo) |
| Arm FF-A virtio-msg bus (informative) | [Arm documentation](https://documentation-service.arm.com/static/68f647791134f773ab3f0a7c) |
