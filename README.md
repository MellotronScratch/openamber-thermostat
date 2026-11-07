# OpenAmber Thermostaat

Een ronde draaiknop-thermostaat (WT32S3-21S / ZX2D10GE01R-V4848, ESP32-S3) die via ESP-NOW
praat met het OpenAmber-display op de warmtepomp. De thermostaat toont statussen, temperaturen en foutcodes,
laat je het setpoint en de modus aanpassen en heeft een weekschema met nachtverlaging. Alle andere instellingen
doe je op OpenAmber zelf.

De koppeling aan de OpenAmber-kant staat in OpenAmber zelf
(`src/openamber/thermostat_link/thermostat_link.yaml`, voorlopig alleen in de build voor het Waveshare-display).

## Inhoud

1. [Bestanden](#bestanden)
   - [Een ander scherm of bord](#een-ander-scherm-of-bord)
2. [Hoe de communicatie werkt](#hoe-de-communicatie-werkt)
3. [Wat standaard is en wat zelf gebouwd is](#wat-standaard-is-en-wat-zelf-gebouwd-is)
4. [Koppelen](#koppelen)
5. [Commando's van thermostaat naar OpenAmber](#commandos-van-thermostaat-naar-openamber)
6. [Overzicht van alle waarden](#overzicht-van-alle-waarden)
7. [Bediening en scherm](#bediening-en-scherm)
8. [Ledring](#ledring)
9. [Werken met de LVGL Designer](#werken-met-de-lvgl-designer)
10. [Flashen](#flashen)
11. [Emulator](#emulator)
12. [Instellingen die je kunt aanpassen](#instellingen-die-je-kunt-aanpassen)
13. [Bekende problemen en valkuilen](#bekende-problemen-en-valkuilen)

Wat nog op de planning staat: zie [BACKLOG.md](BACKLOG.md). Hoe de koppeling werkt en welke bestanden erbij horen (ook in OpenAmber): zie [docs/werking.md](docs/werking.md).

## Bestanden

```
openamber-thermostat-ZX2D10GE01R-V4848.yaml  bordbestand: het bestand dat je compileert (pinnen, scherm, backlight, ledring, knop, BME680)
openamber-thermostat.yaml                    generiek deel: sensoren, ESP-NOW, koppelen, setpoints, scripts
openamber-thermostat-emulator.yaml           emulator op de laptop (zie Emulator)
thermostat/ui_designer.yaml                  lettertypen, kleuren en pagina's uit de LVGL Designer
thermostat/ui_scripts.yaml                   scripts die de pagina's bedienen (menu, modus, koppelen, schema, instellingen, ...)
thermostat/ui_fonts.yaml                     icoonfonts (MDI) die alleen de scripts gebruiken
thermostat/ui_tikken.yaml                    vulling en kleurverloop van de tikken rond het setpoint
thermostat/schema.yaml                       weekschema met nachtverlaging
thermostat/emulator_weg.yaml                 haalt voor de emulator weg wat niet op een laptop kan
```

Het bordbestand laadt het generieke deel via `packages:`, en het generieke deel laadt de bestanden uit `thermostat/`.

### Een ander scherm of bord

Alles wat bij het scherm of bord hoort staat in het bordbestand; het generieke deel kent de hardware niet. Voor een ander
scherm maak je een kopie van het bordbestand (bijvoorbeeld `openamber-thermostat-nieuwscherm.yaml`), pas je de hardware aan, en
compileer je dat bestand. Het generieke deel verandert niet.

Het bordbestand moet leveren:

| Wat | Toelichting |
|---|---|
| `esp32:`, `psram:`, `spi:`, `i2c:`, `display:` | chip, geheugen en het schermmodel |
| `lvgl: buffer_size` | past bij het scherm; de Designer-export neemt dit niet over |
| `light` met id `backlight` | backlight; het generieke deel schakelt hem na 60 s uit |
| `light` met id `status_leds` en een effect `Status` | ledring; het aantal leds hoort bij het bord |
| `sensor` met ids `bme_temp`, `bme_hum`, `bme_pres`, `bme_gas` | BME680 of een vergelijkbare sensor |
| draaiknop | bij draaien het script `knob_step` aanroepen met `delta: 1` of `-1` |
| drukknop | korte druk roept `knob_press_short` aan, lange druk (langer dan 0,8 s) roept `knob_press_long` aan |

Het LVGL-ontwerp (`ui_designer.yaml`) is gemaakt voor 480×480 pixels. Heeft het nieuwe scherm een ander formaat, dan moet
dat ontwerp opnieuw gemaakt worden.

De naam in `esphome: name:` (`openamber-thermostat`) staat in het generieke deel en moet voor alle borden gelijk blijven,
omdat OpenAmber de thermostaat daaraan herkent.

Er is geen `secrets.yaml` nodig; er staat niets geheims in de firmware:

- **WiFi**: niet in de firmware. De gebruiker vult zijn WiFi in via de hotspot (zie Eerste keer instellen).
- **API-sleutel**: niet in de firmware (`api: encryption: {}`). Home Assistant stelt hem in bij het toevoegen en de
  thermostaat bewaart hem. OTA biedt daarna dezelfde sleutel aan.
- **Hotspot**: het wachtwoord is openbaar (zie Eerste keer instellen).
- **ESP-NOW**: de sleutel wordt afgeleid uit de koppelcode (zie Koppelen). De basis daarvoor is de substitution
  `thermostat_link_key` in `openamber-thermostat.yaml` (standaard `openamber-thermostat-link`, gelijk aan de standaard
  in OpenAmber); die is openbaar.

## Hoe de communicatie werkt

ESP-NOW is een directe radioverbinding tussen twee ESP32's, zonder router. Hier wordt het gebruikt via
het ESPHome-onderdeel `packet_transport`.

- **Broadcast**: elk apparaat roept zijn waarden naar iedereen in de buurt. Er staan dus geen MAC-adressen
  in de code.
- **Versleuteld**: met een sleutel die beide kanten uit de koppelcode van OpenAmber afleiden (zie Koppelen).
- **Elke waarde heeft een naam** (zoals `tw` of `err`). De zender zegt "ik deel deze sensor onder die naam", de
  ontvanger zegt "ik wil die naam van dat apparaat".
- **Alleen getallen en aan/uit-waarden** gaan over. Een volledige `climate` (modus plus setpoints) kan niet gedeeld worden.
- Een waarde wordt verstuurd zodra hij verandert, en daarnaast regelmatig opnieuw (elke 15 s).
- **Zelfde WiFi-kanaal**: ESP-NOW gebruikt het kanaal van de WiFi-verbinding. Beide apparaten moeten dus op hetzelfde
  kanaal zitten (in een mesh met meerdere accesspoints kan dat verschillen).
- **WiFi-energiebesparing staat uit** (`power_save_mode: none`) aan beide kanten; anders slaapt de radio tussendoor en
  gaan pakketten verloren.

Belangrijk bij de configuratie:

- De bovenste `encryption` bij `packet_transport` geldt alleen voor het versturen. Elke `providers:`-regel heeft
  zijn eigen `encryption` nodig om te kunnen lezen.
- De `name` van een provider moet gelijk zijn aan de `esphome: name:` van het andere apparaat
  (`openamber-thermostat` aan de OpenAmber-kant, `openamber` aan de thermostaatkant).
- `rolling_code_enable` staat aan beide kanten aan: elk pakket heeft een oplopende teller, oudere pakketten worden
  geweigerd. De teller wordt bij het opstarten meteen naar flash geschreven, anders lopen de kanten na een snelle
  herstart uit de pas.

## Wat standaard is en wat zelf gebouwd is

**Standaard ESPHome**

- Het versturen en ontvangen van sensoren en binary sensors (temperaturen, statussen) en de versleuteling.

**Zelf gebouwd**

1. **Setpoints en modus van OpenAmber naar de thermostaat.** OpenAmber leest elke 5 seconden zijn eigen
   `climate_controller` uit en zet de waarden in gewone sensoren (`slo`, `shi`, `mode`). Na een commando van de
   thermostaat gebeurt dat meteen.
2. **Commando's van de thermostaat naar OpenAmber** in één getal (volgnummer, modus, setpoints). Zie hieronder.
3. **Koppelen** met een koppelcode van OpenAmber, met daarnaast de MAC-code van OpenAmber. Zie hieronder.
4. **Foutcodes** (`e1`, `e2`): OpenAmber stuurt elke fout- en waarschuwingsvlag als één bit; de thermostaat maakt er de lijst met actieve codes van.
5. **Tijd** (`tm`): OpenAmber stuurt de tijd mee, zodat het schema ook werkt als de thermostaat geen internet heeft.
6. **Verbindings- en condenscontrole** op de thermostaat zelf.

## Koppelen

Op OpenAmber staat de koppeling standaard uit. Zet onder Instellingen → Thermostaat → Algemeen (pagina 2) de schakelaar
**Draadloze thermostaat** aan (of de entiteit met die naam). OpenAmber maakt dan een willekeurige **koppelcode van
6 cijfers**; die staat onder de schakelaar en op de webpagina als "Thermostaat koppelcode". Met de schakelaar uit
draait ESP-NOW niet.

- **Op de thermostaat**: Menu → Koppelen (of vanzelf na het opstarten als er nog geen code is). Draai om een cijfer te
  kiezen, druk voor het volgende cijfer. Na het zesde cijfer zoekt de thermostaat de OpenAmber met die code; binnen 30 s
  verschijnt "Gekoppeld met OpenAmber …". Dat kan even duren, want OpenAmber stuurt zijn waarden elke 15 s opnieuw.
- **Sleutel**: beide kanten leiden de ESP-NOW-sleutel af uit de code: SHA-256 van `thermostat_link_key` + ":" + code.
  Zonder de juiste code kan een apparaat niets lezen of versturen. De vaste `thermostat_link_key` is alleen de basis
  (en de sleutel zolang er nog geen code is), zodat er nooit onversleuteld verkeer is.
- **MAC-code, eerste wint**: naast de koppelcode stuurt OpenAmber zijn **MAC-code** mee (`pin`: de laatste 6 tekens van
  zijn MAC-adres). Dat is iets anders dan de koppelcode: de koppelcode maakt de sleutel, de MAC-code zegt welke OpenAmber
  het is. De thermostaat legt de eerste OpenAmber die hij met de juiste koppelcode hoort vast (`pair_code`) en gebruikt
  daarna alleen diens waarden. Hij stuurt die MAC-code terug, en OpenAmber neemt alleen commando's en BME-waarden aan van
  de thermostaat die zijn eigen MAC-code terugstuurt (`thermostat_pin_ok`). Het is een extra controle bovenop de sleutel,
  voor het zeldzame geval dat twee OpenAmbers dezelfde koppelcode hebben.
- **Nieuwe code**: met "Nieuw" op OpenAmber (twee keer tikken: "Zeker?") of de knop "Thermostaat nieuwe koppelcode".
  De thermostaat moet daarna opnieuw gekoppeld worden.

Een code van 6 cijfers is met wat rekenwerk te raden uit opgevangen pakketten; hij houdt buren en andere
ESP-NOW-apparaten buiten, geen vastberaden aanvaller.

Waarden zijn floats met 24 bits nauwkeurigheid. Dat is precies genoeg voor 6 hex-tekens (24 bits), daarom is de MAC-code
6 tekens en niet het volledige MAC-adres (48 bits).

## Commando's van thermostaat naar OpenAmber

De thermostaat pakt alles in één getal (`cmd`):

```
waarde = ((volgnummer * 4 + modus) * 100 + laag * 2) * 100 + hoog * 2
```

| Deel | Betekenis |
|---|---|
| `volgnummer` | 1 tot 99; elke nieuwe opdracht krijgt het volgende nummer |
| `modus` | 0 = uit, 1 = verwarmen, 2 = koelen, 3 = ongewijzigd laten |
| `laag`, `hoog` | setpoints in graden, met stappen van 0,5 (daarom keer 2) |

OpenAmber pakt dat uit en voert het uit. Daarbij gelden deze regels:

- **Zelfde volgnummer als de vorige keer**: negeren. Een herhaald pakket wordt zo niet twee keer uitgevoerd.
- **Andere thermostaat** (stuurt niet de MAC-code van deze OpenAmber terug): negeren. Een thermostaat met een andere
  koppelcode komt niet eens door de versleuteling.
- **Setpoint**: net als op het scherm van OpenAmber wordt alleen het setpoint van de actieve modus gezet:
  bij koelen het hoge (`hoog`), anders het lage (`laag`). Alleen tussen 5 en 35 graden.
- De thermostaat wacht 600 ms na de laatste klik voordat hij verstuurt, zodat snel draaien niet vele commando's geeft.

Zolang je aan de knop draait (en 3 seconden erna), negeert de thermostaat de setpoints die OpenAmber terugstuurt, anders
springt het scherm heen en weer.

Op OpenAmber moet de thermostaatkeuze op **Intern** staan, anders doet de warmtepomp niets met de setpoints.
Setpoints die de thermostaat zet, zijn net als op het scherm van OpenAmber tijdelijk: bij een herstart van OpenAmber
gelden weer de standaardwaarden.

## Overzicht van alle waarden

Van OpenAmber naar de thermostaat:

| Naam | Betekenis |
|---|---|
| `pin` | MAC-code van OpenAmber (laatste 6 hex-tekens van het MAC-adres) |
| `tw` | tapwatertemperatuur |
| `tws` | doeltemperatuur tapwater |
| `tc`, `tuo`, `tui` | cv-aanvoer (Tc), aanvoer (Tuo), retour (Tui) |
| `ta` | buitentemperatuur |
| `rt` | ruimtetemperatuur (Tr) |
| `slo`, `shi` | laag (verwarmen) en hoog (koelen) setpoint |
| `mode` | modus als ESPHome-getal: 0 uit, 2 koelen, 3 verwarmen |
| `cf` | compressorfrequentie (Hz) |
| `e1`, `e2` | actieve fouten en waarschuwingen, één bit per code (zie hieronder) |
| `wsp` | huidig setpoint: gewenste aanvoertemperatuur (stooklijn, handmatig of koelen) |
| `tm` | tijd: minuten sinds maandag 00:00 |
| `err`, `warn` | fout actief, waarschuwing actief |
| `wv`, `cd` | warmtevraag, koudevraag |
| `twa`, `legp` | tapwater aanwezig, legionellapreventie ingeschakeld |
| `twp`, `twv`, `da` | tapwaterpomp, tapwatervraag, tapwater bezig |
| `lega` | legionellacyclus actief |
| `dfr` | ontdooicyclus actief |
| `bu` | backupelement aan |

Van de thermostaat naar OpenAmber:

| Naam | Betekenis |
|---|---|
| `pin` | MAC-code van de gekoppelde OpenAmber, terug als bevestiging |
| `bt`, `bh`, `dp` | temperatuur, luchtvochtigheid en dauwpunt van de BME680 op de thermostaat |
| `cmd` | commando (zie hierboven) |

**Foutcodes `e1` en `e2`**: elke fout- of waarschuwingsvlag van OpenAmber is één bit; 24 bits passen precies in een
waarde. `e1` bit 0-23: P03 P05 P08 P09 P10, F01-F07 F09 F10 F13-F18 F21 F22, E01 E02. `e2` bit 0-20: E03-E08, de
OpenAmber-fouten 401-406 (pomp-, compressor- en klep-timeouts, modbus, failsafe binnendeel), de waarschuwingen
P01 P02 P06 P07 P11 P13 en 407-409. Die volgorde is ook de volgorde van belang: de statusregel toont de eerste
actieve code, de pagina Foutcodes alle (draaien bladert). De volgorde is het protocol: alleen achteraan toevoegen.

## Bediening en scherm

**Draaien** = kiezen of instellen, **korte druk** = openen of bevestigen, **lange druk** = terug naar Temperatuur
(op de schemapagina een stap terug). Onderaan elke pagina staat een hint met wat draaien en drukken daar doen.

| Pagina | Wat je er doet |
|---|---|
| Temperatuur | setpoint instellen: bij koelen het hoge, anders het lage (10 tot 30 graden, stappen van 0,5); korte druk opent het menu |
| Menu | carrousel met Temperatuur, Overzicht, Schema, Foutcodes, Modus, Koppelen en Instellingen |
| Overzicht | temperaturen en statussen van OpenAmber (alleen lezen) |
| Schema | per weekdag een dag- en nachttemperatuur met tijden; kopiëren naar alle dagen |
| Foutcodes | alle actieve fouten en waarschuwingen van OpenAmber, met de tekst uit OpenAmber; draaien bladert erdoorheen |
| Modus | Verwarmen, Koelen of Uit |
| Koppelen | de koppelcode van OpenAmber invoeren |
| Instellingen | Ledring en Helderheid (voorlopig niet beschikbaar), Thema, Sensor, Software-update |

Zo zien de pagina's eruit (uit de emulator, met de voorbeeldwaarden van de nagebootste OpenAmber):

<table>
  <tr>
    <td align="center"><img src="docs/screenshots/OpenAmber-Thermostat_Thermostaat.png" width="200" alt="Temperatuur"><br>Temperatuur</td>
    <td align="center"><img src="docs/screenshots/OpenAmber-Thermostat_Menu.png" width="200" alt="Menu"><br>Menu</td>
    <td align="center"><img src="docs/screenshots/OpenAmber-Thermostat_Overzicht.png" width="200" alt="Overzicht"><br>Overzicht</td>
    <td align="center"><img src="docs/screenshots/OpenAmber-Thermostat_Schema.png" width="200" alt="Schema"><br>Schema</td>
  </tr>
  <tr>
    <td align="center"><img src="docs/screenshots/OpenAmber-Thermostat_Foutcodes.png" width="200" alt="Foutcodes"><br>Foutcodes</td>
    <td align="center"><img src="docs/screenshots/OpenAmber-Thermostat_Modus_Verwarmen.png" width="200" alt="Modus: Verwarmen"><br>Modus: Verwarmen</td>
    <td align="center"><img src="docs/screenshots/OpenAmber-Thermostat_Modus_Koelen.png" width="200" alt="Modus: Koelen"><br>Modus: Koelen</td>
    <td align="center"><img src="docs/screenshots/OpenAmber-Thermostat_Modus_Uit.png" width="200" alt="Modus: Uit"><br>Modus: Uit</td>
  </tr>
  <tr>
    <td align="center"><img src="docs/screenshots/OpenAmber-Thermostat_Koppelen_1.png" width="200" alt="Koppelen: code invoeren"><br>Koppelen: code invoeren</td>
    <td align="center"><img src="docs/screenshots/OpenAmber-Thermostat_Koppelen_2.png" width="200" alt="Koppelen: gekoppeld"><br>Koppelen: gekoppeld</td>
    <td align="center"><img src="docs/screenshots/OpenAmber-Thermostat_Instellingen.png" width="200" alt="Instellingen"><br>Instellingen</td>
    <td></td>
  </tr>
</table>

- **Scherm**: gaat 60 seconden na de laatste bediening uit. De eerste druk of draai maakt het alleen wakker.
- **Pagina-geheugen**: zonder bediening blijft de pagina staan; na 5 minuten gaat hij terug naar Temperatuur.
- **Schema**: stuurt alleen het verwarmingssetpoint, en alleen in de modus Verwarmen.
- **Sensor** (Instellingen): welke kamertemperatuur het scherm toont, Tr van OpenAmber of de eigen BME680.
  OpenAmber regelt altijd op Tr.
- **Thema** (Instellingen): achtergrondverloop en accentkleur (Paars, Oranje, Groen, Blauw, Grijs).
- **Software-update** (Instellingen): toont de geïnstalleerde en de nieuwste versie op GitHub. De knop zoekt naar een
  update, of installeert hem als er een nieuwere is; de thermostaat herstart daarna vanzelf. Hij zoekt ook zelf elke 6 uur.
  In Home Assistant staat dezelfde update als entiteit "Firmware".

**Statusiconen** (Temperatuur en Overzicht): de actieve modus (voorrang: ontdooien, legionella, tapwater, koelen,
verwarmen, standby), backupelement, fout (rood) of waarschuwing (oranje), en geen verbinding als er de afgelopen minuut
geen pakket van de gekoppelde OpenAmber binnenkwam.

## Ledring

De ring heeft 13 adresseerbare leds en het effect `Status` (in het bordbestand): kleur van de modus, ademen als de
compressor draait, een korte flits bij bediening; met het scherm uit alleen rood bij een fout of oranje bij een waarschuwing.

**Voorlopig niet beschikbaar.** Het effect verstuurt de leds elke 40 ms, ook als de kleur niet verandert. Daardoor
geeft het scherm glitches en valt de thermostaat van de WiFi. Zolang `ledring_beschikbaar` op `"false"` staat, blijft
de ring uit en staan Ledring en Helderheid grijs in Instellingen.

## Werken met de LVGL Designer

`ui_designer.yaml` is een export uit de LVGL Designer (https://lvgl.espboards.dev). Let bij elke export op:

1. **Alles binnen het Designer-blok wordt bij een export opnieuw gegenereerd.** Wijzigingen die je met de hand daarin
   maakt, gaan verloren; eigenschappen die de Designer niet kent, worden weggegooid.
2. **`buffer_size`** staat in het bordbestand onder `lvgl:` en niet in het Designer-bestand. Zo gaat hij niet verloren.
3. **Icoonfonts en het achtergrondverloop** worden bij het opstarten op de widgets gezet (`ui_fonts.yaml` en
   `ui_scripts.yaml`), zodat ze een export overleven.
4. **Hernoem je een widget**, pas dan ook `ui_scripts.yaml` en `openamber-thermostat.yaml` aan; die verwijzen met id naar de widgets.

## Flashen

De kant-en-klare firmware staat bij de [releases](https://github.com/MellotronScratch/openamber-thermostat/releases),
per scherm en versie:

```
openamber-thermostat-ZX2D10GE01R-V4848-0.0.1.factory.bin   eerste keer, via USB
openamber-thermostat-ZX2D10GE01R-V4848-0.0.1.ota.bin       update via WiFi
```

- **Eerste keer of bij een kapotte bootloader**: via USB, met het `factory`-bestand (bijvoorbeeld via web.esphome.io).
  Het `ota`-bestand heeft geen bootloader en maakt het apparaat onbruikbaar als je het op die manier flasht.
- **Daarna**: draadloos, via ESPHome Builder (Install, Wirelessly) of `esphome run <bestand.yaml> --device <ip>`.
  Een al gecompileerde build upload je met `esphome upload <bestand.yaml> --device <ip>`; die compileert niet.
- **Volgorde bij wijzigingen aan beide kanten**: eerst OpenAmber, dan de thermostaat.
- De `web_server` staat uit; die gebruikt veel geheugen.

### Eerste keer instellen

1. Zonder WiFi start de thermostaat binnen een minuut de hotspot **OpenAmber Therm Fallback Hotspot**, wachtwoord
   **`openamber`**.
2. Verbind met je telefoon of laptop met de hotspot. Er opent vanzelf een pagina (anders: ga naar http://192.168.4.1) met
   de WiFi-netwerken in de buurt. Kies je netwerk, vul het wachtwoord in en klik op Save.
3. De thermostaat maakt verbinding met je WiFi en bewaart de gegevens; ze blijven staan bij updates. Is je WiFi een keer
   niet te bereiken, dan start de hotspot weer.
4. In Home Assistant verschijnt de thermostaat als nieuw ESPHome-apparaat. Bij het toevoegen stelt Home Assistant de
   API-sleutel in.
5. Koppel de thermostaat met OpenAmber (zie Koppelen).

Let op: zolang er geen API-sleutel is ingesteld, en daarna zonder versleuteling, neemt OTA een update aan van iedereen
op je netwerk. Dat is bij de meeste ESPHome-firmware zo.

### Een release maken

1. Verhoog `version` onder `esphome: project:` in `openamber-thermostat.yaml` (bijvoorbeeld `"0.0.2"`) en commit.
2. Zet een tag met hetzelfde nummer en push hem: `git tag v0.0.2` en `git push origin v0.0.2`.
3. GitHub bouwt de firmware (`.github/workflows/release.yml`, met ESPHome 2026.9.1) en maakt een release met de
   bestanden. Klopt de tag niet met de versie in de yaml, dan stopt de build.

Bij elke release hoort per scherm een manifest, `openamber-thermostat-<bord>.manifest.json`, met de versie en de link
naar het `ota`-bestand. De nieuwste staat altijd op
`https://github.com/MellotronScratch/openamber-thermostat/releases/latest/download/openamber-thermostat-<bord>.manifest.json`;
daar kijkt de update-pagina.

Een nieuw scherm voeg je toe aan `BOARDS` in de workflow; de naam is het deel achter `openamber-thermostat-` in de
naam van het bordbestand, en moet gelijk zijn aan de substitution `bord` in dat bordbestand (daarmee kiest de
update-pagina het juiste manifest).

## Emulator

Met de emulator probeer je het scherm uit op je laptop (ESPHome `host` met een SDL-venster). Nodig: SDL2
(macOS: `brew install sdl2`).

```
esphome run openamber-thermostat-emulator.yaml
```

Pijltje links/rechts = draaien, Enter = korte druk, Backspace = lange druk. De emulator bootst OpenAmber na met vaste
voorbeeldwaarden voor de sensors en statussen (blok `interval:` in het emulatorbestand); setpoints en modus niet. De BME680
geeft ook vaste waarden. Het bordbestand wordt niet gebruikt; `thermostat/emulator_weg.yaml` haalt weg wat niet
op een laptop kan (WiFi, OTA, ESP-NOW, internetklok, sha256).

## Instellingen die je kunt aanpassen

Bovenin `openamber-thermostat.yaml` en `thermostat/ui_scripts.yaml`, onder `substitutions:`:

| Naam | Standaard | Betekenis |
|---|---|---|
| `thermostat_link_key` | openamber-thermostat-link | basis voor de ESP-NOW-sleutel, gelijk aan die van OpenAmber |
| `ledring_beschikbaar` | false | ledring aan of uit te zetten in Instellingen (zie Ledring) |
| `condens_marge` | 2 | waarschuwen als de aanvoer minder dan zoveel graden boven het dauwpunt komt (bij koelen) |
| `pagina_terug_min` | 5 | minuten zonder bediening voordat het scherm terugkeert naar Temperatuur |

**Condens**: alleen bij koelen, als de aanvoertemperatuur (Tuo) minder dan `condens_marge` graden boven het dauwpunt van de
kamer (gemeten door de BME680) komt. De uitkomst staat in de globale `condens_warn`; er is nog geen icoon op het scherm.

**Verbinding**: de thermostaat telt als verbonden zolang er in de afgelopen 60 seconden een pakket van de gekoppelde OpenAmber kwam.

## Bekende problemen en valkuilen

- **Zwart of grijs scherm na ESPHome 2026.10.1.** Op die versie bleef het scherm leeg (ook met het oorspronkelijke bestand),
  terwijl 2026.9.1 het goed deed. De oorzaak is niet gevonden; er is geen melding in de changelog over `mipi_rgb`.
  Bouw het project daarom met 2026.9.1, of test een nieuwere versie eerst met een minimaal bestand dat alleen een kleurvlak toont.
- **Ledring**: geeft glitches en WiFi-uitval, zie Ledring.
- **Eerst `factory`, dan `ota`**: zie Flashen.
- **Namen kloppen niet**: bij "Unknown key"-meldingen eerst controleren of de provider-naam gelijk is aan de `esphome: name:`
  van het andere apparaat, en of `rolling_code_enable` aan beide kanten gelijk staat.
- **Geen data na het flashen**: staat de schakelaar "Draadloze thermostaat" op OpenAmber aan? Klopt de koppelcode (opnieuw
  koppelen)? Hebben beide kanten dezelfde `thermostat_link_key`?
  Draait OpenAmber wel de nieuwe firmware? Bij een crash tijdens het opstarten gaat OpenAmber vanzelf terug naar de vorige versie.
- **Het pad in `packages:`** is relatief aan de map van het hoofdbestand. "No such file or directory" betekent bijna altijd
  een verkeerd pad of een verkeerde bestandsnaam.
- **Logniveau**: zet `espnow` niet op `DEBUG` of `VERBOSE`; dat geeft te veel logregels en kost geheugen. `lvgl` alleen aan als je een schermprobleem zoekt.
