# v380-lan-protocol

Open documentation of the native LAN protocol of V380 (Macro-Video) Wi-Fi
cameras, established on a dual-lens pan/tilt camera sold as **S300**
(hardware `HsAkQQVL`, firmware `AppEV3L_V2_V1.0.9.10_20250731`).

It covers what an open client needs to use the camera locally, without the
V380 Pro app or the vendor cloud: authentication, live HEVC video, audio in
both directions, pan/tilt, configuration reads, plus statically recovered
layouts for live settings and SD-card replay.

See [PROTOCOL.md](PROTOCOL.md). Each section says whether it was validated on
a real camera or only read from the shipped app.

Listed on [OpenDeviceDB](https://opendevicedb.org/) as `macro-video/s300`.

## License

Documentation under [CC BY 4.0](LICENSE). It is an independent write-up for
interoperability; it contains no firmware, vendor code, credentials or
captures.
