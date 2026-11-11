# zt9101-wifi

Working folder for the ZTOP ZT9101 USB Wi-Fi dongle (Speedlink 300 Mbps, `2310:1a2b` -> `2310:9086`).

- `dobbsbp/`            clone of github.com/dobbsbp/ztop-wifi-driver (Debian 13, kernel 6.12); built here
- `windows-package/`    Windows driver from Microsoft Update Catalog (`.cab`, extracted `x/`: .inf names the chip)
- `catalog-html/`       raw catalog pages used to find the download link

Flow and traps: walpad.8g/.agents/shared/memory/usb-dongle-identify.md

## Monitor-mode patch (tested in a Kali VM, kernel 7.2.15, 2026-11-12)

`patches/radianceadm-zt9101-monitor-mode.patch` applies to RadianceADM/zt9101-linux at commit `e656128`
(`git apply`). It fixes, in the driver's monitor path:
1. `-DCONFIG_WIFI_MONITOR`: the wiphy never advertised monitor (macro mismatch with `CFG_ENABLE_MONITOR_MODE`).
2. `_set_monitor_channel` was an empty stub: now retunes the radio (`zt_hw_info_set_channel_bw`).
3. The chip's RX filter was never opened: entering monitor sends `UMSG_OPS_MP_SET_PRX` with AAP|APM|AM|AB|ADF|ACF|AMF.
4. Radiotap always claimed an FCS (readers cut 4 bytes): flag removed.
5. Radiotap channel was the connected network's: now the tuned monitor channel (`nic_info_st.mon_channel`).
6. Radiotap signal was always 0: now `signal_strength - 100` dBm (same as scan results).
7. Monitor-mode TX leaked the skb: now freed.

Results: 15x managed<->monitor + channel changes, 0 failures; monitor while UP works; capture sees beacons, probe,
auth, EAPOL, RTS/CTS/ACK/BA with real dBm; normal Wi-Fi (WPA2 to a 2.4 GHz router) works before and after.
NOT working: packet injection (monitor TX is dropped); `iw dev info` shows no channel line (no `get_channel` op);
no DHCP test (dhclient absent on Kali); 2.4 GHz only. Pre-existing WARNs in the connect path (mlme/auth/assoc).

## Injection (tested in the same Kali VM, 2026-11-12) -- WORKS

`patches/radianceadm-zt9101-monitor-and-injection.patch` supersedes the monitor-only patch above
(applies cleanly from the same RadianceADM commit `e656128`). Beyond the 7 monitor-mode fixes it adds:

8. **`work_monitor_tx_entry`** (`ndev_linux.c`): the vendor stub was `//work_monitor_tx_entry(...)`,
   never implemented. Strips the radiotap header userspace tools (aireplay-ng, mdk4, a raw
   AF_PACKET socket) prepend, and hands the raw 802.11 frame to `zt_nic_mgmt_frame_xmit()` -- the
   driver's own real management-frame TX path (already used for probe requests, auth, assoc).
   `cfg80211`'s `Mgmt_Tx` op looked like the natural place for this but is a dead end: it only
   actually transmits when P2P is active, otherwise it just logs and returns success without
   sending anything.
9. **`.get_channel` op** (`zt_cfg80211.c`, kernel >=6.13 where cfg80211 added the link_id param):
   the wiphy never reported its channel back (`iw dev info` showed none). Needed so tools that
   check "is my netlink channel sane" (aireplay-ng's own channel probe) don't bail immediately --
   though aircrack-ng's *own* ioctl-based channel check still needs `--ignore-negative-one`
   regardless (a well-known aircrack-ng quirk on nl80211-only monitor interfaces, unrelated to
   this driver).
10. **The real blocker, found only by bisecting with a raw `printk` at the top of
    `ndev_start_xmit` and a live `journalctl -k -f` tail**: the vendor driver ties the *kernel*
    netif TX queue state to "associated to an AP" -- `zt_os_api_disable_all_data_queue()` on
    disconnect calls `netif_tx_stop_all_queues()` + drops carrier, and nothing re-enables them for
    monitor mode (monitor never associates, so the normal re-enable on connect-success never
    fires). With the queues left stopped, `dev_queue_xmit()`'s qdisc layer drops every injected
    frame *before* `ndo_start_xmit` is ever called -- independent of anything in
    `work_monitor_tx_entry`. A raw socket's `send()` still reports success either way, because
    that only confirms qdisc enqueue, not actual transmission, which makes this failure mode
    invisible to the most obvious check. Fix: call the driver's own
    `zt_os_api_enable_all_data_queue(ndev)` (same helper the connect-success path uses) when
    entering monitor mode.
    `ip -s link show` is **not** a useful success signal for any of this: the driver overrides
    `.ndo_get_stats` to return `tx_info`/`rx_info`-derived counters (`tx_pkts`, not the
    management-frame-specific `tx_mgnt_pkts` injected frames actually go through), ignoring
    whatever gets written to the stock `net_device.stats`.

**Verification:** `aireplay-ng --ignore-negative-one --deauth 4 -a <router> -c <client>` against
my own router+client reported 64/64 ACKs received (its own over-the-air confirmation, via
listening for real 802.11 ACK frames in response). A concurrent `tcpdump` on the same interface
captured the client doing a full real reconnect (open-system auth -> association -> WPA handshake)
twice during the attack window. Regression: 15x managed<->monitor cycles with channel changes,
0 failures; normal WPA2 connect + ping still 0% loss before and after; no new kernel
warnings/oopses (same 5 pre-existing connect-path WARNs as every earlier run, nic/mlme.c,
nic/auth.c, nic/assoc.c, nic/wlan_mgmt.c -- not touched by any of this work).

**Not done / caveats:** data-frame injection (ARP replay, fragmentation attacks) goes through the
same `zt_nic_mgmt_frame_xmit` path and *should* work the same way (the chip's TX descriptor
doesn't care about 802.11 frame type), but only deauth was actually exercised. 2.4 GHz only.

## Full aircrack-ng suite validation (2026-11-12, same Kali VM)

- **`airmon-ng start wlan0` does not work for this driver**: it prints a confusing, incorrect
  "(mac80211 monitor mode already enabled ... on [phy0]10)" and leaves the interface in
  `type managed`. There is no stray interface; `airmon-ng`'s detection heuristics are built for
  mac80211 drivers, and this is a from-scratch cfg80211 implementation that merely looks similar.
  **Use the manual sequence instead** (confirmed reliable throughout this whole validation):
  `ip link set wlan0 down; iw dev wlan0 set type monitor; ip link set wlan0 up; iw dev wlan0 set channel N`.
  This is not something to fix in the driver or in airmon-ng; just don't rely on airmon-ng here.
- **`airodump-ng`'s own stdout, redirected to a log file, can fill a tmpfs `/tmp` fast**: its
  live ncurses redraw writes hundreds of escape-coded bytes per refresh; three ~30s runs each
  produced 600-700MB of log and filled a 2GB `/tmp` tmpfs ("No space left on device" on an
  unrelated later command). Don't redirect its stdout to a file across repeated runs; `>/dev/null`
  or let it inherit the terminal.
- **Capture a real handshake, end to end**: `airodump-ng -w ... --bssid <AP> -c <ch>` + a single
  `aireplay-ng --ignore-negative-one --deauth 1 -a <AP> -c <client>` (the "1" is round count; each
  round itself bursts up to 64 frames -- that is normal aircrack-ng behaviour on every driver, not
  specific to this one) reliably produces real, over-the-air-acknowledged deauth (confirmed via
  aireplay-ng's own ACK counter) and a captured WPA handshake.
- **A capture with several different, real clients' handshakes mixed together can confuse
  `aircrack-ng`'s per-BSSID handshake tracking** (it does not always pick the best/complete one
  when multiple stations' exchanges interleave) -- this is an aircrack-ng/tshark tooling nuance,
  not a driver bug. Isolating one station's frames (`tshark -Y "wlan.fc.type_subtype==0x08 ||
  wlan.addr==<client>" -F pcap -w isolated.cap` -- the `-F pcap` matters, tshark defaults to
  pcapng, which classic `aircrack-ng` rejects with "Unsupported file format") and cracking that
  isolated capture is the reliable way to validate a specific handshake.
- **Verified by actually cracking a captured handshake**: isolated one station's complete 4-message
  exchange (matched M1/M2 and M3/M4 replay-counter pairs, confirmed via
  `tshark -Y eapol -T fields -e wlan_rsna_eapol.keydes.msgnr -e eapol.keydes.replay_counter`),
  ran `aircrack-ng -w wordlist.txt -b <AP> isolated.cap` with a wordlist containing the network's
  real (owner-supplied, not written here) passphrase among decoys -- **KEY FOUND, matching the
  real network passphrase**. This is full, unambiguous proof the capture is byte-for-byte correct
  (the MIC verifies against the real PSK) and that the whole chain -- monitor-mode RX,
  injection-forced reconnect, and standard `aircrack-ng` tooling -- works together on this dongle
  with the patch applied.
- **Side finding, not related to this work**: one real device on the owner's actual Wi-Fi network
  (`60:01:94:3d:6d:f3`, likely the "WLED-AP" smart-lighting controller seen in scans, given the
  adjacent MAC `62:01:94:3d:6d:f3`) is stuck in an endless Message-1/Message-2 retry loop against
  "Loff" and never completes a handshake -- consistent with a cached wrong Wi-Fi password on that
  device. Worth telling the owner; unrelated to the dongle or any driver change here.
# zt9101
# zt9101
# zt9101
# zt9101
