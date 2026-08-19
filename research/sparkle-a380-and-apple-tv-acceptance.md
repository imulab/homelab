# Sparkle Arc A380 identity and Apple TV Jellyfin acceptance

Research date: 2026-08-11

## Decision summary

- The likely card is the **Sparkle Intel Arc A380 GENIE 6GB**, part number **SA380G-6G**. It is low-profile and ships with a short bracket, but Sparkle specifies a **dual-slot** cooler. No current first-party Sparkle listing matches all three descriptions "A380", "low-profile", and literal "single-slot".
- Do not finalize the physical-install plan until the box/card label or a clear photo confirms the SKU. If it is SA380G-6G, the chassis needs two adjacent low-profile expansion positions plus clearance and airflow.
- Use **Swiftfin** for the Apple TV end-to-end acceptance test. Jellyfin currently labels it **Official Beta**; the current App Store release is free, supports Apple TV, and requires tvOS 26.0 or later.
- Treat **Infuse** as an optional third-party playback/library acceptance client, not the sole hardware-transcoding test: its broad format support tends to avoid server-side video transcoding.
- Prove hardware acceleration twice: first with a deterministic bitrate-limited Jellyfin Web stream, then with Swiftfin on Apple TV. In both cases require Jellyfin Dashboard to report **Transcode** and confirm Intel GPU activity or the Intel hardware path in the FFmpeg log.

## Card identification and installation facts

Sparkle's current A380 catalog contains the compact A380 ELF and the low-profile A380 GENIE. The GENIE is the only likely match for a 2U low-profile installation, but its official specifications conflict with a literal single-slot description.

| Property | Sparkle A380 GENIE |
| --- | --- |
| Product / part number | Sparkle Intel Arc A380 GENIE 6GB / `SA380G-6G` |
| Form factor | Low-profile; short bracket included; **2 slots** |
| Dimensions | **145 x 75 x 35 mm** |
| Interface | **PCIe 4.0 x8**; motherboard requirement is one PCIe 3.0/4.0/5.0 graphics slot |
| Total board power | **75 W TBP** |
| Auxiliary power | **None required**; slot-powered |
| PSU guidance | 350 W or greater |

Primary sources: [Sparkle A380 GENIE product page](https://www.sparkle.com.tw/en/A380-GENIE), [Sparkle SA380G-6G specification sheet](https://www.sparkle.com.tw/files/20240729114753160.pdf), and [Sparkle launch note](https://www.sparkle.com.tw/en/media-center/view/130954BF6CC1).

The closest Sparkle product that really is both low-profile and single-slot is the **A310 ECO**, part `SA310C-4G`, not an A380. This makes a SKU/photo check material rather than cosmetic. Source: [Sparkle A310 ECO specification sheet](https://www.sparkle.com.tw/files/20240202113531413.pdf).

### BIOS implication

Jellyfin's current Intel guide says Resizable BAR is recommended, but not mandatory, for Arc A-series cards such as the A380; it is mandatory for Arc B-series. That means an unsupported firmware modification should not be made merely to add ReBAR to this older platform. The guide also recommends ASPM where the board supports it to reduce Arc idle power. Source: [Jellyfin Intel hardware-acceleration guide](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/).

## Apple TV client choices

### Swiftfin: recommended for acceptance

- Jellyfin lists Swiftfin as **Official Beta**, open source, and available for iOS and Apple TV. Its stated design goal is maximizing direct play with VLC while providing a native Apple interface.
- The current US App Store snapshot is Swiftfin 1.5, released July 16, 2026. It is free, supports Apple TV, and requires **tvOS 26.0 or later**. Verify the household Apple TV can run tvOS 26 before choosing it as the acceptance device.
- Current Swiftfin releases include an app-wide bitrate limit and playback information. Set the app-wide playback bitrate/quality below a test file's known source bitrate to request a controlled transcode. Confirm the exact setting label/path on the installed 1.5 build because UI wording can change.

Primary sources: [Jellyfin client catalog](https://jellyfin.org/downloads/clients/all), [Swiftfin repository](https://github.com/jellyfin/Swiftfin), [Swiftfin releases](https://github.com/jellyfin/Swiftfin/releases), and [Swiftfin App Store listing](https://apps.apple.com/us/app/swiftfin/id1604098728).

### Infuse: optional normal-playback client

- Jellyfin classifies Infuse as **Third Party** and proprietary. Firecore documents a native Jellyfin integration with library access, server metadata/artwork, watched state and progress, remote streaming, collections, and playlists.
- Direct Mode fetches server data on demand and suits large libraries. Library Mode caches/indexes locally; Firecore recommends its optional InfuseSync plugin for Jellyfin when using that mode.
- Infuse is free with optional Pro features. As of this research date, the US storefront shows **$1.99 monthly**, **$16.99 yearly**, or **$99.99 lifetime** for Pro; locale and tax can change the amount. Firecore says Pro purchases include future major updates and support Family Sharing.
- The storefront describes Pro as adding more video formats, high-resolution Dolby/DTS audio, cloud services, broader AirPlay/Cast support, and cross-device synchronization. Jellyfin connectivity is listed as a base streaming capability, but Firecore does not publish a precise free-versus-Pro codec matrix. Test representative media before deciding the free tier is sufficient.
- Because Infuse is built to play a broad range of formats directly, successful Infuse playback may bypass Jellyfin video transcoding entirely. That is desirable for daily viewing but poor as the only proof of the new GPU path.

Primary sources: [Jellyfin client catalog](https://jellyfin.org/downloads/clients/all), [Firecore Jellyfin integration guide](https://support.firecore.com/hc/articles/360006462093-Streaming-from-Plex-Emby-and-Jellyfin), [Firecore purchase options](https://support.firecore.com/hc/en-us/articles/360046954753-Purchases-and-Family-Sharing), and [Infuse App Store listing](https://apps.apple.com/us/app/infuse/id1136220934).

## Acceptance test

Use a source whose bitrate is comfortably above the imposed client limit. A normal H.264 or HEVC title is preferable to an intentionally exotic file because it isolates bitrate-driven transcoding from unrelated compatibility failures.

1. In Jellyfin Web on a laptop or phone, start the test title and select a playback quality/bitrate below the source bitrate.
2. In the Jellyfin Dashboard, require the playback type to be **Transcode**, not Direct Play, Remux, or Direct Stream. Jellyfin defines only Transcode as video-stream transcoding.
3. During playback, confirm Arc use with `intel_gpu_top` on the TrueNAS host, if available, or inspect Jellyfin's FFmpeg log/command for the configured Intel QSV/VA-API hardware path. Also require stable playback and a transcode speed above 1x.
4. Repeat on Apple TV using Swiftfin's app-wide bitrate/quality setting below the same file's source bitrate. Again require Dashboard **Transcode** plus Intel GPU/log evidence.
5. Finally, play representative high-bitrate, HDR, audio, and subtitle combinations in the intended daily client. If using Infuse, this is the direct-play and library/user-experience check, not a replacement for steps 1-4.

Jellyfin explicitly recommends lowering resolution or bitrate to force a video transcode during Intel HWA verification. Unsupported video codecs and subtitle burn-in can also cause a video transcode, but they are less controlled acceptance inputs because client capability profiles and subtitle behavior vary. Sources: [Jellyfin Intel hardware-acceleration guide](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/), [Jellyfin transcoding definitions](https://jellyfin.org/docs/general/post-install/transcoding/), and [Jellyfin codec and subtitle behavior](https://jellyfin.org/docs/general/clients/codec-support/).

## Information still needed

- Photo of the GPU box/card label or exact SKU, especially because "A380 low-profile single-slot" conflicts with Sparkle's current A380 specifications.
- Apple TV model and current tvOS version, to confirm Swiftfin 1.5 compatibility.
- Whether Infuse is already licensed; it is optional for this rollout and should not block GPU acceptance.

## Addendum: China-market A380 Pioneer and X11SSM-F BIOS

Follow-up research date: 2026-08-11

### Revised card identification

The physical description now has a much stronger match: **SPARKLE Intel Arc A380 Pioneer**, part **`SA380P-6G`**. Sparkle's China-market datasheet calls it a China-exclusive, half-height, single-fan, single-slot A380 and specifies **156 x 69 x 15 mm**, **222 g**, **PCIe 4.0 x8**, EAN **4711342290634**, and a short bracket in the box. This reconciles the reported 2U/low-profile and single-slot form factor; it is a different regional product from the global-market A380 GENIE `SA380G-6G`, which is 145 x 75 x 35 mm and dual-slot.

The power figure remains unresolved. The official Pioneer `SA380P-6G` datasheet says **75 W typical board power**, while the user's retail box reportedly says **50 W TDP**. No first-party 50 W revision of the A380 Pioneer was found. Sparkle's officially documented 50 W, low-profile, single-slot product is the **A310 ECO**, not the A380. Therefore, do not overwrite the user's observed label with the web specification, but do not plan around a 50 W A380 until a photo of the SKU/EAN and power label confirms whether this is a regional revision, packaging error, or another product. Source: [Sparkle China A380 Pioneer `SA380P-6G` datasheet](https://sparklecomputer.com/wp-content/uploads/2025/08/SPARKLE%E6%92%BC%E4%B8%8E%E7%A7%91%E6%8A%80-%E8%8B%B1%E7%89%B9%E5%B0%94%E9%94%90%E7%82%AB%E2%84%A2-A380-%E5%85%88%E9%94%8B.pdf); comparison sources: [Sparkle A380 GENIE `SA380G-6G` datasheet](https://www.sparkle.com.tw/files/20240729114753160.pdf) and [Sparkle A310 ECO `SA310C-4G` datasheet](https://www.sparkle.com.tw/files/20240202113531413.pdf).

### X11SSM-F: documented BIOS controls

For an Arc card used headlessly while retaining the ASPEED/IPMI console, the documented board controls support this conservative configuration:

- Keep the onboard VGA hardware enabled: motherboard jumper **JPG1 pins 1-2** is the default onboard-VGA-enabled position. The BMC enable jumper **JPB1 pins 1-2** is also the default enabled position.
- In BIOS, use **Advanced > PCIe/PCI/PnP Configuration > VGA Priority > Onboard**. `Offboard` would instead prioritize an add-in graphics device. Keeping `Onboard` is the documented setting aligned with retaining ASPEED/IPMI as the primary console; the manual does not state that this prevents the operating system from using the Arc as a secondary, headless accelerator.
- In that same menu, the exact path is **Advanced > PCIe/PCI/PnP Configuration > Above 4G Decoding > Enabled**. The manual exposes `Disabled` and `Enabled`; enabling it is appropriate for the Arc setup, while Resizable BAR itself is not documented by this board manual.
- Do not change CSM or boot mode merely to make the Arc headless. The manual documents **Security > Secure Boot Menu > CSM Support** and **Boot > Boot Mode Select** with `Legacy`, `UEFI`, and `Dual` (default `Dual`), but provides no first-party statement that headless Arc use requires disabling CSM or forcing UEFI. Preserve the currently working TrueNAS boot mode unless a reproducible boot problem supplies a reason to change it.
- Do not apply a blanket ASPM setting. The CPU-connected **Slot 6/Slot 7 PEG** menus do not expose a per-slot ASPM enable in this manual. The **PCH Slot 4/Slot 5** menus do expose ASPM and L1-substate controls, so availability depends on the physical slot used. **Program PCIe ASPM After OPROM** controls when ASPM is programmed; it is not an ASPM enable switch. Record the Arc's chosen slot before deciding whether any ASPM control applies.

These menu and jumper details come from the [Supermicro X11SSM-F/X11SSL-F user manual revision 1.1a](https://www.supermicro.com/manuals/motherboard/C236/MNL-1785.pdf): JPG1 and JPB1 on pages 60-61; PCIe slot controls on pages 83-86; Above 4G Decoding and VGA Priority on pages 88-89; CSM Support on page 107; and Boot Mode Select on page 109. Supermicro's current download page lists BIOS **3.6** and explicitly advises against flashing firmware unless there is a firmware-related issue, so a BIOS update should not be treated as a default installation step. Source: [Supermicro X11SSM-F firmware download page](https://www.supermicro.com/en/support/resources/downloadcenter/firmware/MBD-X11SSM-F/BIOS?mlg=0).

### Installation-plan consequence

Proceed on the physical-fit assumption only after checking the card label for **`SA380P-6G`** or EAN **4711342290634**. For the first boot, retain ASPEED/IPMI (`JPG1` enabled and `VGA Priority: Onboard`), enable Above 4G Decoding, and otherwise leave CSM/boot mode and ASPM unchanged. After TrueNAS boots, verify the Arc appears as a separate PCI device before adding it to the Jellyfin container. Treat the card as potentially **75 W slot-powered** for power and thermal planning until the 50 W label discrepancy is resolved.
