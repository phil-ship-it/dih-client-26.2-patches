# DIH Client — Patches für Minecraft 26.2

Patches für [dih-client](https://github.com/ahigherdesire/dih-client) v5.0.1, der offiziell
Minecraft 26.2 (Fabric 0.19.3, Java 25) unterstützt. Getestet auf Fabric Loader 0.19.3.

Die Basis ist GPL-3.0-lizenziert. Diese Patches stehen unter derselben Lizenz.

## Anwendung

```bash
git clone https://github.com/ahigherdesire/dih-client.git
cd dih-client
git apply /pfad/zu/patches/dih-26.2-fixes.patch
./gradlew build
```

Benötigt JDK 25. Das Build-Skript erwartet keinen festen `org.gradle.java.home` —
gegebenenfalls `JAVA_HOME` setzen:

```bash
JAVA_HOME=/pfad/zu/jdk25 ./gradlew build
```

Ergebnis: `build/libs/DIH Client-<version>-26.2-dev.jar`

## Die vier Änderungen

### 1. Layout des Serverlisten-Bildschirms

`src/main/java/dihclient/mixin/DihMultiplayerScreenMixin.java`

Die obere Button-Leiste überlappte sich mit fremden Buttons, zuletzt mit einem
schmalen Vanilla-Button, der mitten in der Leiste klebte.

Ursache war eine fehlerhafte Breitenberechnung:

```java
int topCell = Math.max(1, (topAvailable - GAP * 4) / 5);
```

Die Leiste wurde in fünf Zellen geteilt, belegt wurden aber nur vier Buttons.
Die daraus berechnete Zellbreite wurde anschließend nie verwendet — die Buttons
blieben bei ihren festen 60 px und liefen deshalb über den verfügbaren Platz hinaus.

Zwei weitere Punkte:

- **Minecraft 26.2 hat keinen „Servers"-Button oben.** Der Navigationsbutton heißt
  `Join Server` (Übersetzungsschlüssel `selectServer.select`) und liegt im Footer
  bei `y = height - 52`. Ein Matcher auf `"servers"` kann dort nichts finden.
- Die Leiste wird jetzt rechtsbündig gesetzt, und der freie Platz ergibt sich aus
  den **gemessenen** Kanten aller sichtbaren Fremd-Buttons derselben Zeile
  (`widget.getX() + widget.getWidth()`), statt aus einer statischen Annahme.

### 2. Tote Proxies werden automatisch deaktiviert

`src/main/java/dihclient/util/DihProxyManager.java`

`DihProxy.isValid()` prüfte ausschließlich die Syntax — Adresse nicht leer, Port im
gültigen Bereich. Ein Proxyserver, der die Verbindung ablehnt, galt damit als gültig.
`getEnabled()` gab ihn zurück, jeder Join endete in `Connection refused`.

`getEnabled()` führt jetzt einen TCP-Verbindungstest mit 2 s Timeout durch und
deaktiviert unerreichbare Proxyserver. Ergebnisse werden 60 Sekunden zwischengespeichert,
damit der Test nicht bei jedem Renderframe erneut läuft.

### 3. Farbschema auf Catppuccin Mocha

`src/main/java/dihclient/util/DihConfig.java`
`src/main/java/dihclient/util/DihTheme.java`

Damit DIH zum Catppuccin-Addon in Meteor passt, wurde das blaue Standardschema
durch die Mocha-Palette ersetzt. Die semantischen Farben bleiben erhalten
(`.success` grün, `.danger` rot).

| Kanal | Vorher | Nachher | Catppuccin |
|---|---|---|---|
| `master`, `accent` | `#3B6EFF` | `#CBA6F7` | mauve |
| `outline` | `#2B50B3` | `#585B70` | surface2 |
| `text` | `#F3ECE7` | `#CDD6F4` | text |
| `toggle` | `#3B6EFF` | `#89B4FA` | blue |
| `backdrop` | `#4863B2` | `#313244` | surface0 |
| `success` | `#35D873` | `#A6E3A1` | green |
| `danger` | `#E26A6A` | `#F38BA8` | red |
| `button` | `#1F368F` | `#45475A` | surface1 |
| `header` | `#3B6EFF` | `#B4BEFE` | lavender |
| `hover` | `#6482FF` | `#94E2D5` | teal |

`DihTheme.DEFAULTS` enthält die Quellfarben, aus denen die UI-Texturen eingefärbt
werden, in derselben Reihenfolge wie `Channel`. Auch diese Palette wurde
angepasst, sonst wären Texturen blau geblieben.

**Wichtig:** Wer das Update bereits verwendet hat, muss zusätzlich
`config/dih/config.json` anpassen. DIH speichert die Farben persistent; die
gespeicherten Werte überschreiben sonst die neuen Defaults. Eine angepasste
Fassung liegt unter `config/`.

### 4. Feldbreite in der Top-Leiste

`src/main/java/dihclient/mixin/DihMultiplayerScreenMixin.java`

Nach Behebung des Überlappens wurden die Buttons zu schmal und schnitten ihre
Beschriftungen ab (`Proxies` → `roxie`, `Accounts` → `unts`). Ursache war eine
zusätzliche Deckelung:

```java
int topCell = Math.max(1, Math.min(maxCell, stackW / buttonCount));
```

`stackW / buttonCount` ergab 48 px — weniger als die benötigten 60 px. Die
Breite ist jetzt `Math.max(Mindestbreite, verfügbarerPlatz / buttonCount)`,
sodass keine Beschriftung mehr abgeschnitten wird.

## Inhalt

```
patches/dih-26.2-fixes.patch   alle vier Änderungen als ein Patch
config/config.json              ThemeColors auf Catppuccin Mocha gesetzt
docs/kit-funktion-26.2.md       korrigierte provipvp:kit-Funktion
README.md                       dieses Dokument
```

## Nicht Bestandteil dieser Patches

Einige Beobachtungen aus dem Betrieb mit 26.2, die nicht Teil des DIH-Quellcodes sind:

- **`provipvp:kit`** stürzte beim Laden ab. In 26.2 ist `ItemEnchantments` eine
  flache Map (`unboundedMap(ENCHANTMENT, LEVEL_CODEC)`) ohne `levels`-Wrapper.
  Siehe `docs/kit-funktion-26.2.md`.
- **Zwei Baritones kollidieren.** DIH bettet ein vollständiges eigenes Baritone
  ein. Ein separates `baritone-fabric-26.2.jar` (`baritone-fork`) führt zu
  `NoSuchMethodError` auf `IEventBus.onReceiveChatMessage`, weil DIH diese
  Methode in sechs Klassen aufruft. Das separate Baritone muss deaktiviert werden.
- **AUTISM Client** ist im Wesentlichen dieselbe Codebasis. Zusammen mit DIH
  ergibt das rund 230 identische Mixins und damit einen abstürzenden Loader.
- **Meteor-Addons** sind für 26.2 weitgehend abgebrochen. Von 17 verifiziert
  26.2-fähigen Addons waren 16 bereits installiert.
- **`mc-traffic-control`** wirkt beim Start einen `NoClassDefFoundError` auf
  `com.moulberry.lattice.element.LatticeElements`. Lattice hat keine
  26.2-Version. Das Mod selbst funktioniert, nur sein ModMenu-Eintrag nicht.

## Lizenz

GPL-3.0, wie die Basis. Siehe `LICENSE`.
