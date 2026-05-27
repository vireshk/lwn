# Virtio goes message-based: introducing virtio-msg

Virtio has long relied on **virtio-pci** and **virtio-mmio**: the driver configures
devices by reading and writing memory-mapped transport registers.
**virtio-msg** replaces that register traffic with structured **messages** while
keeping the same device model—features, configuration space, virtqueues, and
standard device types such as net and block.

Virtio Over Messages (aka virtio-msg) lives in the at [Linaro's
fork](https://github.com/Linaro/virtio-msg-spec/tree/virtio-msg) of the Virtio
specification. Bill Mills posted an earlier revision to
`virtio-comment@lists.linux.dev` in January 2026.

The work sits under Linaro's **HVAC** (Heterogeneous VirtIO for Automotive
Computing) umbrella, with contributors from Linaro, Arm, AMD, ST, Google, and
others. The goal is not to replace virtio-pci or virtio-mmio where those work
well, but to give Virtio a transport that fits software-only peers,
heterogeneous SoCs, secure partitions, and multi-system links where
trap-and-emulate register access is awkward or impossible.

---

## When PCI and MMIO are the wrong abstraction

Existing transports assume a **register window** the driver can access directly:
feature bits, status, config space, queue addresses, and notifications map to
fixed offsets.

That assumption fails when both sides are software but the link is not a guest
PCI function or MMIO device—for example:

- A **co-processor** (MCU, DSP, firmware) on an AMP SoC, reached via shared
  memory and doorbells.
- **Two machines** on PCIe, each with its own OS.
- **Normal and secure worlds** using an IPC ABI such as Arm FF-A.
- **Host-to-guest, or guest-to-guest** paths where a hypervisor exposes messages
  rather than emulated virtio-mmio.
- **Userspace backends** in the same OS as the driver, with nothing to trap on.

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

## Control plane vs data plane

**Control plane (messages):** Discovery, feature negotiation, configuration,
virtqueue setup, device status (`DRIVER_OK`, reset), and transport-visible
notifications are **request/response pairs and events** defined in the spec.

**Data plane (virtqueues):** Bulk I/O still uses **standard virtqueues** in
memory (shared memory or other DMA-accessible memory, as with other transports).
Transport messages configure and signal the rings; they do not carry virtqueue
descriptor traffic. Optional **shared-memory regions** (`GET_SHM`) describe
additional device-owned memory; drivers may query them, but use is optional.

virtio-msg is a new **transport binding**, not a new device protocol.

---

## Two layers: transport and bus

| Layer | Responsibility |
|-------|----------------|
| **Transport** | Normative per-device messages (features, config, queues, status, events). Identical on every bus. |
| **Bus** | Delivers message buffers between endpoints; discovery; memory sharing; hotplug; interpretation of addresses in `SET_VQUEUE`. |

Device drivers (virtio-net, virtio-blk, …) use the **transport** protocol. A **bus
instance** underneath may use FF-A, shared-memory AMP, Xen grants, or other
carriers. The spec's goal is that virtio device implementations do not need
virtio-msg-specific logic beyond what any transport requires.

### Transport messages

Transport messages use stable IDs (revision 1), for example:

| ID | Message | Initiator |
|----|---------|-----------|
| 0x02 | `GET_DEVICE_INFO` | Driver |
| 0x03–0x08 | Features, config, status | Driver |
| 0x09–0x0B | `GET_VQUEUE`, `SET_VQUEUE`, `RESET_VQUEUE` | Driver |
| 0x0C | `GET_SHM` | Driver |
| 0x40–0x42 | `EVENT_CONFIG`, `EVENT_AVAIL`, `EVENT_USED` | Device or driver (see spec) |

**Devices must implement** all core request/response messages including `GET_SHM`.
**Drivers must** support the messages needed to initialize and operate a device;
`GET_SHM` is optional on the driver side.

### Initialization

After the bus exposes a **device number**, initialization aligns with core
Virtio, but each step is a message:

1. **`GET_DEVICE_INFO`** — the only message allowed as an **early-discovery
   exception** before the normal status sequence. The response reports device
   type, vendor ID, a **16-byte device UUID** (nil UUID if none), **feature block**
   count, config size, `max_virtqueues` (up to 65536, including any admin
   virtqueues), and optional **admin virtqueue** range (`admin_vq_start`,
   `admin_vq_count`).
2. **Reset and status** — `SET_DEVICE_STATUS` through ACKNOWLEDGE and DRIVER.
3. **Feature negotiation** — `GET_DEVICE_FEATURES` / `SET_DRIVER_FEATURES` in
   32-bit blocks (device-offered bits vs driver-selected subset); verify
   `FEATURES_OK` via status. **Transport feature bits** (e.g. strict config) come
   from the bus instance, not from these device feature messages.
4. **Config and queues** — `GET_CONFIG` / `SET_CONFIG` under the active config
   profile; `SET_VQUEUE` / `GET_VQUEUE` for each queue.
5. **`DRIVER_OK`** — normal virtqueue I/O may begin.

The spec requires devices **not** to offer `VIRTIO_F_NOTIF_CONFIG_DATA` on this
transport. `VIRTIO_F_NOTIFICATION_DATA` may be negotiated for `EVENT_AVAIL`
encoding when the bus can preserve those semantics.

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

### Common message format

Every message starts with an **8-byte header**:

- **`type`:** bit 0 = request (0) or response (1); bit 1 = transport (0) or bus (1).
- **`msg_id`:** bits 0–5 = message number; bit 6 = event (1) vs request/response (0);
  bit 7 = implementation-defined (1) vs standardized (0).
- **`dev_num`:** target device for transport messages; **0** for bus messages.
- **`token`:** opaque to driver and device; the **bus** may rewrite it for
  correlation; the device **copies** the request token into the response before
  any bus rewrite.
- **`msg_size`:** total length including header.

Little-endian throughout. Buses must forward transport payloads unchanged except
for token handling and generic limits (such as maximum message size).

### Ordering, errors, and completion

The spec separates **requests** (exactly one protocol response or a
**transport-visible failure** from the bus) from **events** (one-way; may be
dropped unless a bus-specific rule applies). For each driver/device pair, the
device processes requests in **receive order**; the bus may have multiple
outstanding requests and may deliver responses **out of order** if it correlates
them with `token`. Events may interleave with request/response traffic.

Malformed or unsupported messages are discarded without synthetic error responses;
drivers use bounded timeouts and recovery via reset/status rules.

### Bus messages and discovery

Optional bus messages (all with `dev_num = 0`):

- **`GET_DEVICES`** — bitmap windows of present device numbers.
- **`PING`** — health check; response echoes request data.
- **`EVENT_DEVICE`** — hotplug with `DEVICE_BUS_STATE_ADDED` (0x0001) or
  `DEVICE_BUS_STATE_REMOVED` (0x0002).

Buses may implement any subset of these; discovery may instead use device tree,
ACPI, or hypervisor data. Buses **must** validate `dev_num` on transport messages,
route opaquely, and **must not** invent fake transport responses on routing failure.

Because `dev_num` is 16-bit, each bus instance supports on the order of 65,000
devices; larger systems use **multiple bus instances**.

**Transport parameters** per bus instance: revision (revision 1 allows 52–65535
byte messages; **264 bytes recommended**), maximum message size, and transport
feature bits. These are stable for an active device unless the bus forces
revalidation.

### Runtime notifications

After `DRIVER_OK`, **polling-only** operation is allowed: if both ends agree, they
may omit `EVENT_AVAIL` and/or `EVENT_USED` and poll queue state instead. When
events are used, the bus may forward them in-band or synthesize equivalent
transport events from out-of-band interrupts—but each notification occurrence
still has one authoritative transport-message path documented by the bus.

`EVENT_AVAIL` carries the queue index and, when `VIRTIO_F_NOTIFICATION_DATA` is
negotiated, packed `next_off` / `next_wrap` in `next_offset`.

---

## Informative bus bindings

The OASIS chapter defines the **common transport**. Concrete bindings are
separate (informative examples in the spec):

| Binding | Role |
|---------|------|
| **Virtio message bus over FF-A** | Normal↔secure, host↔guest, guest↔guest; [Arm FF-A bus document](https://documentation-service.arm.com/static/68f647791134f773ab3f0a7c). |
| **PCI for AMP** | Heterogeneous SoCs or multi-host PCIe; [hvac-demo](https://github.com/wmamills/hvac-demo) proof of concept. |
| **Xen grants and events** | Under discussion (queue address spaces). |

**Future work** mentioned in project materials includes virtio-msg over **admin
virtqueues** on virtio-pci (sub-devices behind an existing PCI virtio function).

HVAC demos (QEMU, open-amp, multi-core SoCs) exercise these bindings; they
informed the spec text but do not define it.

---

## Relative to the January 2026 virtio-comment posting

The public **[PATCH v1 0/4]** series on virtio-comment was an earlier snapshot.
The current `transport-msg.tex` on the `virtio-msg` branch incorporates review
feedback, including among other items:

- **`GET_DEVICE_INFO`** extended with **device UUID**, **feature blocks** (not
  only a bit count), and 32-bit admin-virtqueue fields.
- **`GET_SHM`** responses using **64-bit** address and length and a `shmid` field.
- **Baseline vs strict** configuration profiles and clearer `EVENT_CONFIG` rules.
- **Message ordering**, **error completion**, and **bounded failure** rules for buses.
- **Initialization** sequencing aligned explicitly with core Virtio status flow.
- Full **driver / device / bus** conformance clauses and integration into the
  main specification structure.

A follow-on posting to virtio-comment is expected once review stabilizes.

---

## Why standardize this in Virtio

Without a common transport message set, every FF-A, AMP, or hypervisor project
risks reimplementing feature negotiation, config access, and queue setup
incompatibly. virtio-msg provides one **control-plane protocol** for standard
Virtio devices, while each bus solves delivery, discovery, and memory sharing.

For **automotive and embedded** SoCs (HVAC's motivation), an application processor
can target ordinary virtio device types on a companion core once the **bus
binding** for that link is agreed.

For **secure partitions**, SHARE vs LEND and similar policies stay in the bus
binding; the same transport messages apply in the TEE as in the normal world.

---

## Prototypes (brief)

Reference code exists in several trees (Linux, QEMU, open-amp, TEE and hypervisor
demos). Implementations will drift until the spec revision is accepted upstream;
for protocol details, use **`transport-msg.tex`** on the virtio-spec `virtio-msg`
branch.

---

## References

| Item | Link |
|------|------|
| Virtio spec source (`transport-msg.tex`) | [Linaro/virtio-msg-spec](https://github.com/Linaro/virtio-msg-spec/tree/virtio-msg) — `virtio-msg` branch |
| Earlier virtio-comment cover (Jan 2026) | [lore.kernel.org — Bill Mills](https://lore.kernel.org/all/20260126163230.1122685-1-bill.mills@linaro.org/) |
| Arm FF-A virtio-msg bus (informative) | [Arm documentation](https://documentation-service.arm.com/static/68f647791134f773ab3f0a7c) |
| HVAC overview | [Linaro Confluence](https://linaro.atlassian.net/wiki/spaces/HVAC/overview) |
| Demos (informative) | [hvac-demo](https://github.com/wmamills/hvac-demo) |
