# kali-utm-apple-silicon-troubleshooting

# Troubleshooting: Kali Linux Display & Input Issues in UTM (Apple Silicon)

A troubleshooting log for display scaling, mouse input, and blank-screen issues when running Kali Linux as a guest VM in UTM on an Apple Silicon Mac.

> **Status legend:** ✅ Resolved · ⚠️ Workaround · 🔍 Open

---

## Environment

| Component | Details |
|---|---|
| Host | Apple Silicon Mac (macOS) |
| Hypervisor | UTM (QEMU backend) |
| Guest | Kali Linux (ARM64, Xfce desktop) |
| VM type | "Other" — see [Prior Issue](#prior-issue-boot-errors) |

---

## Summary of Issues

| # | Symptom | Root Cause | Status |
|---|---|---|---|
| 1 | VM display stuck at a small resolution; no way to maximize | SPICE guest agent not installed | ✅ |
| 2 | Mouse clicks don't register / miss targets | Guest and host cursor positions out of sync | ✅ |
| 3 | Display blank on VM startup after config changes | Display card / saved resolution | ⚠️ |
| 4 | Display goes blank when window is resized mid-session | Dynamic resolution change fails in guest | ⚠️ |

---

## Issue 1 — Small Display, Can't Maximize

**Symptom:** Kali boots to a small fixed-resolution window. Resizing or maximizing the UTM window doesn't change the guest resolution.

**Root cause:** Without the SPICE guest agent, Kali can't communicate with UTM about the window size. UTM's Linux guest documentation lists `spice-vdagent` as required for dynamic display resolution (QEMU backend) and clipboard sharing.

**Fix:**

```bash
sudo apt update
sudo apt install spice-vdagent qemu-guest-agent
sudo reboot
```

Then, with the VM shut down, in **UTM → VM Settings → Display**:
- Display card: `virtio-gpu-pci`
- Enable **Resize display to window size automatically**

**Interim workaround:** Pick a larger resolution manually in Kali's **Settings → Display**.

**Result:** ✅ Display resized after installing the agent.

---

## Issue 2 — Mouse Clicks Not Registering

**Symptom:** Clicking on UI elements inside the VM does nothing or lands in the wrong place. Changing the mouse option in the UTM toolbar didn't help.

**Root cause:** Same as Issue 1. The SPICE agent handles client mouse mode, which keeps the guest cursor in sync with the host. Without it, the guest's idea of the cursor position drifts from where the host cursor is.

**Fix:**
1. Install `spice-vdagent` (Issue 1).
2. In **UTM → VM Settings → Input**:
   - **USB Support:** `USB 3.0` (or 2.0) — required; gives the VM a USB controller for the virtual pointer.
   - **USB Sharing:** *Not required.* This only passes physical USB devices (flash drives, Wi-Fi adapters) from the Mac into the VM.
   - Disable any option that forces a PS/2 controller.
3. If clicks are offset, toggle **Retina mode** in Display settings.

**Fallback:** Use the **mouse capture** button in the UTM toolbar to send input directly to the VM. Press **Control + Option** to release. (Documented in UTM's Windows 11 guide, but the control applies to any guest.)

**Result:** ✅

---

## Issue 3 — Blank Display on Startup

**Symptom:** After the display configuration changes above, the VM window stays black on boot.

**Troubleshooting steps (in order):**

1. **Switch the display card.** Shut down → Display → `virtio-ramfb`. `virtio-gpu-pci` may show nothing until the desktop loads; `ramfb` displays from early boot. Avoid `-gl` (GPU-accelerated) variants, which often go black with Kali.
2. **Wait.** The login screen can take 30–60 seconds; a keypress or mouse movement may wake the display.
3. **Reset a bad saved resolution.** If the blank screen followed a resolution change in Kali:
   ```bash
   # Switch to a text console: Ctrl + Option + F2 (add fn if needed)
   rm ~/.config/monitors.xml
   sudo reboot
   ```

**Result:** ⚠️ _Fill in which step resolved it._

---

## Issue 4 — Display Blanks on Resize Mid-Session

**Symptom:** Dragging the UTM window to resize it while Kali is running turns the display black, and it doesn't recover.

**Root cause:** Dynamic resolution changes during a session can fail in the guest. The Xfce compositor is a common contributor.

**Option A — Fixed resolution (most stable):**
1. Shut down → Display → **disable** "Resize display to window size automatically."
2. Set a fixed resolution in Kali:
   - 13–14" MacBook: `1440×900` or `1680×1050` (fallback `1280×800`)
   - 16" MacBook Pro / external monitor: `1920×1080`
3. Use UTM's zoom/full-screen controls instead of dragging the window edge.
4. If text looks tiny, turn **Retina mode** off (it renders at double density).

**Option B — Keep auto-resize, disable compositing:**
Kali **Settings → Window Manager Tweaks → Compositor** → uncheck **Enable display compositing**, then retest resizing.

**Recovery without rebooting the VM:**
```bash
# Ctrl + Option + F2, log in, then:
sudo systemctl restart lightdm
```
> ⚠️ Restarting the display manager closes open apps — unsaved work is lost.

**Result:** ⚠️ _Fill in which option you kept._

---

## Prior Issue: Boot Errors

When creating the VM, selecting **"Other"** instead of **"Linux"** as the VM type fixed persistent boot errors. Found through the official UTM docs and GitHub community discussions.

---

## Key Takeaways

- **Install guest tools first.** Most display and input problems in UTM Linux guests trace back to a missing `spice-vdagent`.
- **USB Support ≠ USB Sharing.** One is the VM's USB controller (needed for the pointer); the other is device passthrough (optional).
- **Dynamic resolution trades stability for convenience.** A fixed resolution is the reliable baseline; auto-resize is a nice-to-have.
- **Keep a TTY escape hatch.** `Ctrl + Option + F2` plus `systemctl restart lightdm` recovers a dead display without a hard reset.
- **Check primary docs.** UTM's official guest-support pages confirmed the agent requirement. Settings-level fixes (display card, compositor) came from AI-assisted troubleshooting and were verified by testing.

---

## Sources

| Source | Used For |
|---|---|
| [UTM Docs — Linux Guest](https://docs.getutm.app/guest-support/linux/) | SPICE agent required for dynamic resolution & clipboard |
| [UTM Docs — Guest Support](https://docs.getutm.app/guest-support/guest-support/) | Overview of guest tools |
| [UTM Docs — Settings Reference](https://docs.getutm.app/settings-qemu/devices/display/) | Dynamic resolution / Retina display options |
| [UTM Docs — Windows 11 Guide](https://docs.getutm.app/guides/windows/) | Mouse capture mode (Control + Option to release) |
| [utmapp/vd_agent (GitHub)](https://github.com/utmapp/vd_agent) | How the SPICE agent handles client mouse mode & auto-resize |
| AI-assisted troubleshooting (Claude) | Display card choice, compositor fix, TTY recovery, resolution reset |
