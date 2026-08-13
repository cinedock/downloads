# CineDock Downloads

Official downloads and installation information for CineDock.

## Desktop beta 4.2.0

| Platform | Download | Requirements |
|---|---|---|
| Windows | [CineDock Desktop for Windows](https://github.com/cinedock/downloads/releases/download/desktop-v4.2.0-beta.1/CineDock-Desktop-4.2.0-Windows-x64.exe) | Windows 10 or 11, 64-bit Intel/AMD PC |
| macOS | [CineDock Desktop for Intel Mac](https://github.com/cinedock/downloads/releases/download/desktop-v4.2.0-beta.1/CineDock-Desktop-4.2.0-macOS-Intel.zip) | Intel `x86_64` Mac; tested on macOS Monterey 12.7.6 |

The desktop apps run CineDock locally and connect to an existing Plex, Emby or Jellyfin server. Docker is not required for desktop use.

These beta builds are not yet code-signed. Windows may show a Microsoft SmartScreen warning. macOS will show an unidentified-developer warning: verify the supplied SHA-256 checksum, extract the ZIP, then Control-click `CineDock.app`, choose **Open**, and confirm. Never disable Gatekeeper.

[Release notes and checksums](https://github.com/cinedock/downloads/releases/tag/desktop-v4.2.0-beta.1)

## Television app 0.2.2

[Download CineDock TV](https://github.com/cinedock/downloads/releases/download/v0.2.2/CineDock-TV.apk)

SHA-256:

```text
A0CE3FF96895B03791F8AB74214F4681EFA0B33EDDAD11284D47CBFF328488F2
```

The APK is release-signed for Amazon Fire TV and NVIDIA Shield TV. Future CineDock TV updates will use the same signing identity.

The CineDock server must already be installed and configured in a normal browser. Open the TV app, enter that CineDock server address—for example, `http://192.168.1.50:8945`—and choose **Open CineDock**. The media-server account and optional TMDB key stay on the CineDock server; they do not need to be entered again on the television.

Full illustrated Amazon Fire TV and NVIDIA Shield TV instructions are available from **Settings → Watching on a TV** inside CineDock. If the wrong server address is entered, press the television remote's **Menu** or **Settings** button to reopen the address screen.

## Support

Report installation and app problems through [CineDock support](https://github.com/cinedock/unraid-templates/issues).

## Support development

CineDock is free. If it is useful to you, you can optionally [buy me a coffee](https://buymeacoffee.com/cinedock).
