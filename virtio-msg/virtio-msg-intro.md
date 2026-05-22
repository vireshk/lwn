# Virtio goes message-based: introducing virtio-msg

Virtio has long relied on **virtio-pci** and **virtio-mmio**: the driver configures
devices by reading and writing transport registers, usually through PCI BARs or
memory-mapped I/O. **virtio-msg** keeps the same Virtio device model: feature
bits, configuration space, virtqueues, and standard device types such as net and
block. What changes is the transport: those operations become structured
**messages**.

Virtio Over Messages, also called virtio-msg, currently lives in
[Linaro's fork](https://github.com/Linaro/virtio-msg-spec/tree/virtio-msg) of
the Virtio specification. Bill Mills posted an earlier revision to
`virtio-comment@lists.linux.dev` in January 2026.

The work sits under Linaro's **HVAC** (Heterogeneous VirtIO for Automotive
Computing) umbrella, with contributors from Linaro, Arm, AMD, ST, Google, and
others. The goal is not to replace virtio-pci or virtio-mmio where those work
well, but to give Virtio a transport that fits software-only peers,
heterogeneous SoCs, secure partitions, and multi-system links where
trap-and-emulate register access is awkward or impossible.

---

## When PCI and MMIO are not the optimal abstraction

Existing transports assume a **register window** that the driver can access
directly: feature bits, status, config space, queue addresses, and notifications
map to fixed offsets.

That assumption is a poor fit when both sides are software endpoints but the
link is not naturally a guest PCI function or MMIO device. Examples include:

- A **co-processor** (MCU, DSP, firmware) on an AMP SoC, reached via shared
  memory and doorbells.
- **Two machines** on PCIe, each with its own OS.
- **Normal and secure worlds** using an IPC ABI such as Arm FF-A.
- **Host-to-guest, or guest-to-guest** paths where a hypervisor exposes messages
  rather than emulated virtio-mmio.
- **Userspace backends** in the same OS as the driver, with nothing to trap on.

In all of these cases, driver and device are often **implemented in software on
both sides**, but there is no natural place to hang a virtio-mmio register block
or a virtio-pci BAR. What the system does have is a **message channel**:
hypercalls, FF-A direct messages, shared-memory queues with interrupts, Xen
events and grants, or something similar.

**virtio-msg** is the proposed answer: keep Virtio's device model (features,
config space, virtqueues, standard device types like net and blk), but implement
the **transport layer** by exchanging structured messages instead of MMIO-style
register access.

---

## Control plane vs data plane

**Control plane (messages):** Discovery, feature negotiation, configuration,
virtqueue setup, device status (`DRIVER_OK`, reset), and transport-visible
notifications are **request/response pairs and events** defined in the spec.

**Data plane (virtqueues):** Bulk I/O still uses **standard virtqueues** in
memory (shared memory or other DMA-accessible memory, as with other transports).
Transport messages configure and signal the rings; they do not carry descriptor
traffic or data buffers. Optional **shared-memory regions** (`GET_SHM`) describe
additional device-owned memory. Devices must implement the query, while drivers
use it only when a device type or feature needs such a region.

virtio-msg is a new **transport binding**, not a new device protocol.

---

## Two layers: transport and bus

| Layer | Responsibility |
|-------|----------------|
| **Transport** | Per-device messages for features, config, queues, status, and events. The same message semantics apply on every bus. |
| **Bus** | Carries messages between local and remote endpoints, validates device numbers, discovers or exposes remote devices and hotplug when needed, and defines memory-sharing and address rules such as how `SET_VQUEUE` addresses are interpreted. |

Device drivers (virtio-net, virtio-blk, …) use the **transport** protocol. A **bus
instance** underneath may use FF-A, shared-memory AMP, Xen grants, or other
carriers. The spec's goal is that virtio device implementations share one
transport message set even when the underlying bus binding is completely
different.

### Transport messages

Revision 1 defines the transport message IDs below:

| ID | Message | Initiator |
|----|---------|-----------|
| 0x02 | `GET_DEVICE_INFO` | Driver |
| 0x03–0x08 | Features, config, status | Driver |
| 0x09–0x0B | Virtqueues - Get, set, reset | Driver |
| 0x0C | `GET_SHM` | Driver |
| 0x40–0x42 | Events - config, avail, used | Device or driver |

**Devices must implement** all core request/response messages, including
`GET_SHM`. **Drivers must** support the messages needed to initialize and
operate a device; `GET_SHM` is intentionally absent from the driver's mandatory
core list, because many devices do not expose device-owned shared-memory
regions.

### Initialization

After the bus exposes a **device number**, initialization aligns with core
Virtio, but each step is a message:

1. **`GET_DEVICE_INFO`** — the only message allowed as an **early-discovery
   exception** before the normal status sequence. The response reports device
   type, vendor ID, a **16-byte device UUID** (nil UUID if none), **feature block**
   count, config size, `max_virtqueues` (up to 65536, including any admin
   virtqueues), and optional **admin virtqueue** range (`admin_vq_start`,
   `admin_vq_count`).
2. **Reset and status** — `SET_DEVICE_STATUS` drives reset, ACKNOWLEDGE, and
   DRIVER.
3. **Feature negotiation** — `GET_DEVICE_FEATURES` / `SET_DRIVER_FEATURES` in
   32-bit blocks (device-offered bits vs driver-selected subset); verify
   `FEATURES_OK` via status. **Transport feature bits** (e.g. strict config) come
   from the bus instance, not from these device feature messages.
4. **Config and queues** — `GET_CONFIG` / `SET_CONFIG` under the active config
   profile; `SET_VQUEUE` / `GET_VQUEUE` for each queue.
5. **`DRIVER_OK`** — normal virtqueue I/O may begin.

The spec requires devices **not** to offer `VIRTIO_F_NOTIF_CONFIG_DATA` on this
transport. `VIRTIO_F_NOTIFICATION_DATA` may be negotiated for `EVENT_AVAIL`
encoding only when the bus can preserve those semantics.

### Configuration semantics profiles

Revision 1 defines two **configuration profiles**, selected by a transport
feature bit on the bus instance
(`VIRTIO_MSG_F_STRICT_CONFIG_GENERATION`):

- **Baseline:** drivers send `generation = 0` on `SET_CONFIG`; devices ignore
  request generation and do not reject writes solely for generation mismatch.
- **Strict:** drivers track generation from `GET_CONFIG` / `EVENT_CONFIG` and
  include it on `SET_CONFIG`; devices reject stale generations.

Both profiles require devices to bump **generation** when config may have changed
asynchronously and to use `EVENT_CONFIG` for device-originated config or status
changes that need asynchronous reporting—not for synchronous status writes the
driver already performed via `SET_DEVICE_STATUS`.

For `SET_CONFIG`, the response `length` is also part of the success indication:
a nonzero write is either applied in full and echoed with the requested length,
or not applied and reported with length zero. Partial nonzero writes are not
valid success results.

### Common message format

Every message starts with an **8-byte header**:

- **`type`:** bit 0 = request (0) or response (1); bit 1 = transport (0) or bus
  (1). Events use request type; they are not responses.
- **`msg_id`:** bits 0–5 = message number; bit 6 = event (1) vs request/response (0);
  bit 7 = implementation-defined (1) vs standardized (0).
- **`dev_num`:** target device for transport messages; **0** for bus messages.
  Device number 0 is valid for transport messages.
- **`token`:** opaque to driver and device; the **bus** may rewrite it for
  correlation; the device **copies** the request token into the response before
  any bus rewrite.
- **`msg_size`:** total length including header.

Little-endian throughout. Buses must forward transport payloads unchanged except
for token handling and generic limits (such as maximum message size).

### Ordering, errors, and completion

The spec separates **requests** (exactly one protocol response or a
**transport-visible failure** from the bus) from **events** (one-way; may be
dropped unless a bus-specific rule applies). Traffic for different device
numbers may be interleaved, but for each driver/device pair the bus presents
requests to the device in receive order and the device processes them in that
order. If a bus allows multiple outstanding requests, it uses `token` for
correlation rather than treating response order as the ordering rule. Events may
interleave with request/response traffic.

Malformed or unsupported messages are discarded without synthetic error responses;
drivers use bounded timeouts and recover via the reset/status rules. The spec
does not define a dedicated transport error message; a routing, policy, timeout,
or completion problem is surfaced by the bus as a transport-visible failure for
the original request.

### Bus messages and discovery

Optional bus messages (all with `dev_num = 0`):

- **`GET_DEVICES`** — bitmap windows of present device numbers.
- **`PING`** — health check; response echoes request data.
- **`EVENT_DEVICE`** — hotplug with `DEVICE_BUS_STATE_ADDED` (0x0001) or
  `DEVICE_BUS_STATE_REMOVED` (0x0002).

Buses may implement any subset of these; discovery may instead use device tree,
ACPI, or hypervisor data. Buses **must** validate `dev_num` on transport messages,
route transport payloads opaquely, and avoid transport-message interpretation
beyond generic checks such as message size.

Because `dev_num` is 16-bit, each bus instance supports on the order of 65,000
devices; larger systems can use **multiple bus instances** to expose more
devices.

**Transport parameters** per bus instance: revision (revision 1 allows 52–65535
byte messages; **264 bytes recommended**), maximum message size, and transport
feature bits.

### Runtime notifications

After `DRIVER_OK`, **polling-only** operation is allowed: if both ends agree, they
may omit `EVENT_AVAIL` and/or `EVENT_USED` and poll queue state instead. When
events are used, the bus may forward them in-band or use out-of-band signaling
as a wakeup hint. In the latter case, the receiving side synthesizes the
corresponding event message back to the transport so the transport semantics
still see exactly one notification occurrence.

`EVENT_AVAIL` carries the queue index and, when `VIRTIO_F_NOTIFICATION_DATA` is
negotiated, packed `next_off` / `next_wrap` in `next_offset`.

---

## Informative bus bindings

The draft chapter defines the **common transport**. Concrete bindings are
separate; the spec lists these as informative examples:

| Binding | Role |
|---------|------|
| **Virtio message bus over FF-A** | Normal↔secure, host↔guest, guest↔guest; [Arm FF-A bus document](https://documentation-service.arm.com/static/68f647791134f773ab3f0a7c). |
| **PCI for AMP** | Heterogeneous SoCs or multi-host PCIe; [hvac-demo](https://github.com/wmamills/hvac-demo) proof of concept. |
| **Xen grants and events** | Under discussion (queue address spaces). |

HVAC demos (QEMU, open-amp, multi-core SoCs) exercise these bindings; they
informed the spec text but do not define it. Future work discussed in project
materials includes virtio-msg over **admin virtqueues** on virtio-pci, exposing
message-based sub-devices behind an existing PCI virtio function.

---

## Why standardize this in Virtio

Without a common transport message set, every FF-A, AMP, or hypervisor project
risks reimplementing feature negotiation, config access, and queue setup
incompatibly. virtio-msg provides one **control-plane protocol** for standard
Virtio devices, so drivers and devices can reuse a generic virtio-msg transport
implementation while each bus solves delivery, discovery, and memory sharing.

For **automotive and embedded** SoCs (HVAC's motivation), an application processor
can target ordinary virtio device types on a companion core once the **bus
binding** for that link is agreed.

For **secure partitions**, SHARE vs LEND and similar policies stay in the bus
binding; the same transport messages apply in the TEE as in the normal world.

The useful split is therefore narrow but important: virtio-msg standardizes
the part every implementation would otherwise reinvent, while leaving
bus-specific questions -- endpoint discovery, memory ownership, interrupt
delivery, and address meaning -- to the binding that actually knows the link.

## Collaborative development via Core Collective

Collaborative development on virtio-msg is being done within the Virtualization
Working Group of the [Core Collective](https://corecollective.dev/), which is an
open collaborative forum for the Arm ecosystem. Current discussions at the
Working group can be seen
[here](https://groups.google.com/a/corecollective.dev/g/virtualization).

If you'd like to participate, please email <enquiries@corecollective.dev> to
join.

---

## References

| Item | Link |
|------|------|
| Virtio spec source (`transport-msg.tex`) | [Linaro/virtio-msg-spec](https://github.com/Linaro/virtio-msg-spec/tree/virtio-msg) — `virtio-msg` branch |
| Earlier virtio-comment cover (Jan 2026) | [lore.kernel.org — Bill Mills](https://lore.kernel.org/all/20260126163230.1122685-1-bill.mills@linaro.org/) |
| Arm FF-A virtio-msg bus (informative) | [Arm documentation](https://documentation-service.arm.com/static/68f647791134f773ab3f0a7c) |
| HVAC overview | [Linaro Confluence](https://linaro.atlassian.net/wiki/spaces/HVAC/overview) |
| Demos (informative) | [hvac-demo](https://github.com/wmamills/hvac-demo) |
| Linux | [Kernel](https://git.kernel.org/pub/scm/linux/kernel/git/vireshk/linux.git) — `virtio-msg` branch |
| Linux RFC (2025) | [lore.kernel.org](https://lore.kernel.org/all/cover.1753865268.git.viresh.kumar@linaro.org/) |
