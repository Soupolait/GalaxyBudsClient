> # AI Disclosure
> This fix has been entirely vibe coded (Mistral CLI, GLM 5.3). I only wished to fix issue #640 on *my* laptop, and this worked. I am currently learning character manipulations in Kotlin, so I am very far away from being able to proof-read this fix.
>I only know that it worked for me, and should someone wish for a quick fix before an upstream fix well you're welcome to try !

> [!NOTE]
> This repository is a personal fork of [Galaxy Buds Manager](https://github.com/timschneeb/GalaxyBudsClient) with Linux fixes:
> - Fixes [#640](https://github.com/timschneeb/GalaxyBudsClient/issues/640) ("BlueZ timed out while connecting to device"): after a connection timeout, the app automatically re-attaches to the earbuds as soon as they become available, instead of staying stuck in an error state until manually reconnected.
> - Improves autostart reliability on Linux (tray icon crash race and Bluetooth timeout at login); see `FIX_NOTES.md`.
<p align="center">
  <a href="https://github.com/timschneeb/GalaxyBudsClient/releases">
    <img alt="GitHub downloads count" src="https://img.shields.io/github/downloads/thepbone/galaxybudsclient/total">
  </a>
  <a href="https://github.com/timschneeb/GalaxyBudsClient/releases">
   <img alt="GitHub release (latest by date)" src="https://img.shields.io/github/v/release/thepbone/galaxybudsclient">
  </a>
  <a href="https://github.com/timschneeb/GalaxyBudsClient/blob/master/LICENSE">
      <img alt="License" src="https://img.shields.io/github/license/thepbone/galaxybudsclient">
  </a>
  <a href="https://github.com/timschneeb/GalaxyBudsClient/releases">
    <img alt="Platform" src="https://img.shields.io/badge/platform-Windows/macOS/Linux/Android-yellowgreen">
  </a>
</p>

## Building from source

Requires the [.NET SDK](https://dotnet.microsoft.com/download) (see `global.json` for the pinned version) and git:

```bash
git clone https://github.com/Soupolait/GalaxyBudsClient.git
cd GalaxyBudsClient
dotnet publish -f net10.0 -r linux-x64 -o bin_linux64 -c Release \
  -p:PublishSingleFile=true --self-contained true \
  GalaxyBudsClient/GalaxyBudsClient.csproj
```

The self-contained single-file binary is output to `bin_linux64/GalaxyBudsClient` (no .NET runtime is required on the target machine). For other RIDs (e.g. `linux-arm64`), replace `-r linux-x64`.

If your installed SDK is older than the one pinned in `global.json`, temporarily adjust its `version` field, build, then restore the file.

See [FIX_NOTES.md](FIX_NOTES.md) for the fixes carried by this fork and the systemd autostart workaround. Prebuilt Linux binaries are available under [Releases](https://github.com/Soupolait/GalaxyBudsClient/releases).

## Credits

### Contributors

* [@nift4](https://github.com/nift4) - macOS support and bug fixes
* [@ArthurWolfhound](https://github.com/ArthurWolfhound) - Issue templates, wiki and translations
* [@AndriesK](https://github.com/AndriesK) - Buds Live bug fix
* [@TheLastFrame](https://github.com/TheLastFrame) - Buds Pro icons
* [@githubcatw](https://github.com/githubcatw) - Connection dialog base
* [@GaryGadget9](https://github.com/GaryGadget9) - WinGet package
* [@joscdk](https://github.com/joscdk) - AUR package

### Translators

* [@ArthurWolfhound](https://github.com/ArthurWolfhound) - Russian and Ukrainian translation
* [@PlasticBrain](https://github.com/fhalfkg) - Korean and Japanese translation
* [@cozyplanes](https://github.com/cozyplanes) - Korean translation
* [@corydalis10](https://github.com/corydalis10) - Korean translation
* [@erenbektas](https://github.com/erenbektas) and [@Eta06](https://github.com/Eta06)  - Turkish translation
* [@kakkk](https://github.com/kakkk), [@KevinZonda](https://github.com/KevinZonda), [@ssenkrad](https://github.com/ssenkrad), [@pseudor](https://github.com/pseudor), [@YexuanXiao](https://github.com/YexuanXiao) and [@mediateeee](https://github.com/mediateeee) - Chinese translation
* [@YiJhu](https://github.com/YiJhu) and [@mediateeee](https://github.com/mediateeee) - Chinese-Traditional translation
* [@efrenbg1](https://github.com/efrenbg1) and Andrew Gonza - Spanish translation
* [@giovankabisano](https://github.com/giovankabisano) - Indonesian translation
* [@lucasskluser](https://github.com/lucasskluser) and [@JuanFariasDev](https://github.com/juanfariasdev) - Portuguese translation
* [@alb-p](https://github.com/alb-p), [@mario-donnarumma](https://github.com/mario-donnarumma) - Italian translation
* [@Buashei](https://github.com/Buashei) - Polish translation
* [@KatJillianne](https://github.com/KatJillianne) and [@thelegendaryjohn](https://github.com/thelegendaryjohn) - Vietnamese translation
* [@joskaja](https://github.com/joskaja) and [@Joedmin](https://github.com/Joedmin) - Czech translation
* [@Benni0109](https://github.com/Benni0109), [@TheLastFrame](https://github.com/TheLastFrame), [@timschneeb](https://github.com/timschneeb) - German translation
* [@nikossyr](https://github.com/nikossyr) - Greek translation
* [@grigorem](https://github.com/grigorem) - Romanian translation
* [@tretre91](https://github.com/tretre91) - French translation
* [@Sigarya](https://github.com/Sigarya) - Hebrew translation
* [@domroaft](https://github.com/domroaft) - Hungarian translation
* [@lampi8426](https://github.com/lampi8426) - Dutch translation
* [@bluday](https://github.com/bluday) - Swedish translation

### Services

* [Cloudflare](https://www.cloudflare.com/) - Secures the backend APIs of GalaxyBudsClient and provided a Pro license

### Assets
* Earbud asset used in the Android icon created by [Archival](https://www.flaticon.com/authors/archival) from [Flaticon](https://www.flaticon.com/)

## License

This project is licensed under [GPLv3](https://github.com/timschneeb/GalaxyBudsClient/blob/master/LICENSE). It is not affiliated with Samsung nor supervised by them in any way.

```
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED,
INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR
THE USE OR OTHER DEALINGS IN THE SOFTWARE.
```
