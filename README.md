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

De naam in `esphome: name:` (`openamber-thermostaat`) staat in het generieke deel en moet voor alle borden gelijk blijven,
omdat OpenAmber de thermostaat daaraan herkent.

Benodigd in `secrets.yaml`:

```yaml
wifi_ssid: "..."
wifi_password: "..."
api_key: "..."        # 32 bytes base64, bijvoorbeeld: openssl rand -base64 32
ap_password: "..."    # wachtwoord van de fallback-hotspot
```

De sleutel voor ESP-NOW is de substitution `thermostat_link_key` in `openamber-thermostat.yaml`
(standaard `openamber-thermostat-link`, gelijk aan de standaard in OpenAmber).

## Hoe de communicatie werkt

ESP-NOW is een directe radioverbinding tussen twee ESP32's, zonder router. Hier wordt het gebruikt via
het ESPHome-onderdeel `packet_transport`.

- **Broadcast**: elk apparaat roept zijn waarden naar iedereen in de buurt. Er staan dus geen MAC-adressen
  in de code.
- **Versleuteld**: beide kanten gebruiken dezelfde sleutel (`thermostat_link_key`). Die zit in de openbare firmware en houdt
  dus alleen andere ESP-NOW-apparaten buiten; de koppelcode is de echte controle.
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
  (`openamber-thermostaat` aan de OpenAmber-kant, `openamber` aan de thermostaatkant).
- `rolling_code_enable` staat uit. Met rolling code lopen de tellers van beide kanten uit de pas na een herstart.

## Wat standaard is en wat zelf gebouwd is

**Standaard ESPHome**

- Het versturen en ontvangen van sensoren en binary sensors (temperaturen, statussen) en de versleuteling.

**Zelf gebouwd**

1. **Setpoints en modus van OpenAmber naar de thermostaat.** OpenAmber leest elke 5 seconden zijn eigen
   `climate_controller` uit en zet de waarden in gewone sensoren (`slo`, `shi`, `mode`). Na een commando van de
   thermostaat gebeurt dat meteen.
2. **Commando's van de thermostaat naar OpenAmber** in één getal (volgnummer, modus, setpoints). Zie hieronder.
3. **Koppelen** met een code die uit het MAC-adres komt. Zie hieronder.
4. **Foutcode** (`ec`): OpenAmber zet de eerste actieve fout of waarschuwing om in één getal.
5. **Tijd** (`tm`): OpenAmber stuurt de tijd mee, zodat het schema ook werkt als de thermostaat geen internet heeft.
6. **Verbindings- en condenscontrole** op de thermostaat zelf.

## Koppelen

Waarom: zonder koppelcode zou elke thermostaat met elke OpenAmber in de buurt praten. De code voorkomt dat,
zonder dat er een MAC-adres in de code hoeft te staan.

- **De code** is de laatste 6 tekens van het MAC-adres van OpenAmber, als getal. OpenAmber stuurt die als
  eerste waarde (`pin`) in elk pakket mee. Je vindt het MAC-adres in het wifi-instellingenmenu van OpenAmber.
- De thermostaat bewaart de gekozen code in de tekst-entiteit `pair_code` (blijft bewaard na een herstart).
- **Vergelijking**: bij elk binnenkomend pakket vergelijkt de thermostaat de `pin` met `pair_code`. Alleen bij gelijke
  code (`pin_ok`) worden de waarden gebruikt; alles van een andere OpenAmber wordt genegeerd.
- **Andersom**: de thermostaat stuurt de gekozen code terug, en OpenAmber negeert commando's en
  BME-waarden van een thermostaat met een andere code (`thermostat_pin_ok`).
- **Op de thermostaat kiezen**: Menu → Koppelen toont een lijst van alle OpenAmbers die recent gehoord zijn.
  Draai om te kiezen, druk kort om te bevestigen. Is er nog niet gekoppeld, dan opent de pagina een paar seconden na
  het opstarten vanzelf.

Waarden zijn floats met 24 bits nauwkeurigheid. Dat is precies genoeg voor 6 hex-tekens (24 bits), daarom is de code 6 tekens
en niet het volledige MAC-adres (48 bits).

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
- **Andere thermostaat** (koppelcode klopt niet): negeren.
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
| `pin` | koppelcode (laatste 6 tekens MAC) |
| `tw` | tapwatertemperatuur |
| `tws` | doeltemperatuur tapwater |
| `tc`, `tuo`, `tui` | cv-aanvoer (Tc), aanvoer (Tuo), retour (Tui) |
| `ta` | buitentemperatuur |
| `rt` | ruimtetemperatuur (Tr) |
| `slo`, `shi` | laag (verwarmen) en hoog (koelen) setpoint |
| `mode` | 0 uit, 1 verwarmen, 2 koelen |
| `cf` | compressorfrequentie (Hz) |
| `ec` | foutcode, zie hieronder |
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
| `pin` | koppelcode (de gekozen OpenAmber) |
| `bt`, `bh`, `dp` | temperatuur, luchtvochtigheid en dauwpunt van de BME680 op de thermostaat |
| `cmd` | commando (zie hierboven) |

**Foutcode `ec`**: 1xx is een P-code (P05 wordt 105), 2xx een F-code (F14 wordt 214), 3xx een E-code (E03 wordt 303),
401 tot 406 de overige fouten (bijvoorbeeld modbus-timeout), 407 tot 409 de overige waarschuwingen, 0 betekent niets aan de hand.
Eerst komt de eerste actieve fout, daarna de eerste waarschuwing.

## Bediening en scherm

**Draaien** = kiezen of instellen, **korte druk** = openen of bevestigen, **lange druk** = terug naar Temperatuur
(op de schemapagina een stap terug). Onderaan elke pagina staat een hint met wat draaien en drukken daar doen.

| Pagina | Wat je er doet |
|---|---|
| Temperatuur | setpoint instellen: bij koelen het hoge, anders het lage (10 tot 30 graden, stappen van 0,5); korte druk opent het menu |
| Menu | carrousel met Temperatuur, Overzicht, Schema, Foutcodes, Modus, Koppelen en Instellingen |
| Overzicht | temperaturen en statussen van OpenAmber (alleen lezen) |
| Schema | per weekdag een dag- en nachttemperatuur met tijden; kopiëren naar alle dagen |
| Foutcodes | de eerste actieve fout of waarschuwing van OpenAmber, met de tekst uit OpenAmber |
| Modus | Verwarmen, Koelen of Uit |
| Koppelen | een OpenAmber kiezen uit de lijst |
| Instellingen | Ledring en Helderheid (voorlopig niet beschikbaar), Thema, Sensor, Software-update |

- **Scherm**: gaat 60 seconden na de laatste bediening uit. De eerste druk of draai maakt het alleen wakker.
- **Pagina-geheugen**: zonder bediening blijft de pagina staan; na 5 minuten gaat hij terug naar Temperatuur.
- **Schema**: stuurt alleen het verwarmingssetpoint, en alleen in de modus Verwarmen.
- **Sensor** (Instellingen): welke kamertemperatuur het scherm toont, Tr van OpenAmber of de eigen BME680.
  OpenAmber regelt altijd op Tr.
- **Thema** (Instellingen): achtergrondverloop en accentkleur (Paars, Oranje, Groen, Blauw, Grijs).
- **Software-update** (Instellingen): de pagina is er, maar er is nog geen updatebron.

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

- **Eerste keer of bij een kapotte bootloader**: via USB, met `firmware.factory.bin` (bijvoorbeeld via web.esphome.io).
  Het `ota`-bestand heeft geen bootloader en maakt het apparaat onbruikbaar als je het op die manier flasht.
- **Daarna**: draadloos, via ESPHome Builder (Install, Wirelessly) of `esphome run <bestand.yaml> --device <ip>`.
  Een al gecompileerde build upload je met `esphome upload <bestand.yaml> --device <ip>`; die compileert niet.
- **Volgorde bij wijzigingen aan beide kanten**: eerst OpenAmber, dan de thermostaat.
- De `web_server` staat uit; die gebruikt veel geheugen.

## Emulator

Met de emulator probeer je het scherm uit op je laptop (ESPHome `host` met een SDL-venster). Nodig: SDL2
(macOS: `brew install sdl2`).

```
esphome run openamber-thermostat-emulator.yaml
```

Pijltje links/rechts = draaien, Enter = korte druk, Backspace = lange druk. Er komen geen gegevens van OpenAmber binnen
en de BME680 geeft vaste waarden. Het bordbestand wordt niet gebruikt; `thermostat/emulator_weg.yaml` haalt weg wat niet
op een laptop kan (WiFi, OTA, ESP-NOW, internetklok).

## Instellingen die je kunt aanpassen

Bovenin `openamber-thermostat.yaml` en `thermostat/ui_scripts.yaml`, onder `substitutions:`:

| Naam | Standaard | Betekenis |
|---|---|---|
| `thermostat_link_key` | openamber-thermostat-link | sleutel voor ESP-NOW, gelijk aan die van OpenAmber |
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
  van het andere apparaat, en of `rolling_code_enable` uit staat.
- **Geen data na het flashen**: controleer of beide kanten dezelfde `thermostat_link_key` hebben en of de koppelcode klopt.
  Draait OpenAmber wel de nieuwe firmware? Bij een crash tijdens het opstarten gaat OpenAmber vanzelf terug naar de vorige versie.
- **Het pad in `packages:`** is relatief aan de map van het hoofdbestand. "No such file or directory" betekent bijna altijd
  een verkeerd pad of een verkeerde bestandsnaam.
- **Logniveau**: zet `espnow` niet op `DEBUG` of `VERBOSE`; dat geeft te veel logregels en kost geheugen. `lvgl` alleen aan als je een schermprobleem zoekt.
