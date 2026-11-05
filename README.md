# OpenAmber Thermostaat

Een ronde draaiknop-thermostaat (WT32S3-21S / ZX2D10GE01R-V4848, ESP32-S3) die via ESP-NOW
praat met het OpenAmber-display op de warmtepomp. De thermostaat toont statussen en temperaturen
en laat je het setpoint en de modus aanpassen. Alle andere instellingen doe je op OpenAmber zelf.

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
11. [Instellingen die je kunt aanpassen](#instellingen-die-je-kunt-aanpassen)
12. [Bekende problemen en valkuilen](#bekende-problemen-en-valkuilen)

## Bestanden

```
openamber-thermostat-ZX2D10GE01R-V4848.yaml  bordbestand: het bestand dat je compileert (pinnen, scherm, backlight, ledring, knop, BME680)
openamber-thermostat.yaml                    generiek deel: sensoren, ESP-NOW, koppelen, scripts, besturing
openamber-thermostat-emulator.yaml           emulator op de laptop (SDL-venster), zie CLAUDE.md
thermostat/ui_designer.yaml                  lettertypen, kleuren en pagina's uit de LVGL Designer (wordt vervangen bij een export)
thermostat/ui_scripts.yaml                   scripts die iets met het scherm doen
thermostat/ui_fonts.yaml                     icoonfonts (MDI)
thermostat/ui_tikken.yaml                    tikken rond het setpoint
thermostat/schema.yaml                       nachtverlaging per weekdag
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
- De waarden worden regelmatig opnieuw verstuurd (het interval stelt `packet_transport` in). Of ESPHome een te vol
  pakket zelf over meerdere pakketten verdeelt, is niet getest; zie de opmerking over pakketgrootte onderaan.

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
   `climate_controller` uit en zet de waarden in gewone sensoren (`slo`, `shi`, `mode`). Die gaan dan via het standaardmechanisme mee.
2. **Commando's van de thermostaat naar OpenAmber** in één getal (volgnummer, modus, setpoints). Zie hieronder.
3. **Koppelen** met een code die uit het MAC-adres komt. Zie hieronder.
4. **Foutcode** (`ec`): OpenAmber zet de eerste actieve fout of waarschuwing om in één getal.
5. **Verbindings- en condenscontrole** op de thermostaat zelf.

## Koppelen

Waarom: zonder koppelcode zou elke thermostaat met elke OpenAmber in de buurt praten. De code voorkomt dat,
zonder dat er een MAC-adres in de code hoeft te staan.

- **De code** is de laatste 6 tekens van het MAC-adres van OpenAmber, als getal. OpenAmber stuurt die als
  eerste waarde (`pin`) in elk pakket mee. Je vindt het MAC-adres in het wifi-instellingenmenu van OpenAmber.
- De thermostaat bewaart de gekozen code in de tekst-entiteit `pair_code` (blijft bewaard na een herstart).
- **Vergelijking**: bij elk binnenkomend pakket vergelijkt de thermostaat de `pin` met `pair_code`. Alleen bij gelijke
  code (`pin_ok`) worden de waarden gebruikt; alles van een andere OpenAmber wordt genegeerd.
- **Andersom**: OpenAmber stuurt de thermostaat-code die hij terugkrijgt ook naar zichzelf en negeert commando's en
  BME-waarden van een andere thermostaat (`thermostat_pin_ok`).
- **Op de thermostaat kiezen**: de koppelpagina toont een lijst van alle OpenAmbers die recent gehoord zijn.
  Draai aan de knop om te kiezen, druk kort om te bevestigen. Een lange druk op de knop opent de koppelpagina.
  Is er nog niet gekoppeld, dan opent de pagina een paar seconden na het opstarten vanzelf.

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
- **Setpoints**: alleen toegepast als `laag >= 5`, `hoog <= 35` en `laag <= hoog`.
- De thermostaat wacht 600 ms na de laatste klik voordat hij verstuurt, zodat snel draaien niet vele commando's geeft.

Zolang je aan de knop draait (en 3 seconden erna), negeert de thermostaat de setpoints die OpenAmber terugstuurt, anders
springt het scherm heen en weer.

Op OpenAmber moet de thermostaatkeuze op **Intern** staan, anders doet OpenAmber niets met de commando's.

## Overzicht van alle waarden

Van OpenAmber naar de thermostaat:

| Naam | Betekenis |
|---|---|
| `pin` | koppelcode (laatste 6 tekens MAC) |
| `tw` | tapwatertemperatuur |
| `tws` | doeltemperatuur tapwater |
| `tc`, `tuo`, `tui` | cv-aanvoer (Tc), aanvoer (Tuo), retour (Tui) |
| `ta` | buitentemperatuur |
| `rt` | ruimtetemperatuur |
| `slo`, `shi` | laag en hoog setpoint |
| `mode` | 0 uit, 1 verwarmen, 2 koelen |
| `cf` | compressorfrequentie (Hz) |
| `ec` | foutcode, zie hieronder |
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

De knop heeft vier contexten (`knob_context`):

| Context | Pagina | Draaien doet |
|---|---|---|
| 0 | Home | setpoint aanpassen |
| 1 | Overview | niets |
| 2 | Modus kiezen | momenteel niet bereikbaar |
| 3 | Koppelen | kiezen uit de lijst |

- **Korte druk**: op Home en Overview wisselen tussen de twee; op de koppelpagina bevestigen.
- **Lange druk**: opent de koppelpagina.
- Het scherm en de ledring gaan 60 seconden na de laatste bediening uit en komen terug bij draaien of drukken.

**Overview-pagina**: kamertemperatuur, setpoint en tapwatertemperatuur, plus statusiconen:

- **Hoofdicoon (`overview_icon_mode_heatcooltapleg`)**: toont de actieve modus. Voorrang als er meerdere
  tegelijk gelden: ontdooien, legionellarun, tapwater maken, koelen, verwarmen, standby.
- **Bliksem**: het backupelement staat aan.
- **Uitroepteken**: rood bij een fout, oranje bij een waarschuwing, anders verborgen.
- **Link-uit-icoon**: er is de afgelopen minuut geen pakket van de gekoppelde OpenAmber binnengekomen. Het hoofdicoon, de bliksem en het uitroepteken worden dan verborgen, want die gegevens zijn verouderd.
- **Kraantje naast de tapwatertemperatuur**: alleen zichtbaar als tapwater aanwezig is; blauw als het bezig is, anders grijs.

Een condenswaarschuwing wordt berekend (zie [Instellingen](#instellingen-die-je-kunt-aanpassen)) maar heeft nog geen icoon op het scherm.

## Ledring

De ring heeft 13 adresseerbare leds. Na bediening toont hij hoe vol het tapwatervat is, als een vat dat volloopt:

- Onderaan beginnen leds 8 en 9, dan loopt het aan beide kanten omhoog (7 en 10, 6 en 11, 5 en 12, 4 en 0, 3 en 1)
  en eindigt het bij led 2 bovenaan. De nummering loopt met de klok mee.
- Het niveau volgt de tapwatertemperatuur: leeg bij `vat_leeg` (20 °C), vol bij de doeltemperatuur van OpenAmber (`tws`).
  Geeft OpenAmber geen doeltemperatuur, dan geldt `vat_doel_standaard` (55 °C).
- De kleur hangt af van de positie: blauw onderin, oranje in het midden, rood bovenin.
- Tijdens het opwarmen pulseert de led aan de bovenkant van de vulling.

Ligt de ring anders ingebouwd, pas dan `led_bodem` aan. Het effect `LED test` laat elke led in een eigen kleur zien om de
volgorde te bepalen.

## Werken met de LVGL Designer

`ui_designer.yaml` is een export uit de LVGL Designer en mag je helemaal vervangen. Let bij elke export op:

1. **Ongebruikte lettertypen verwijderen.** Roboto en Material+Icons bovenaan geven de compileerfout
   "Font Material+Icons is missing 1 glyph". Zet ze niet in het bestand.
2. **`buffer_size: 25%`** staat bewust in het hoofdbestand onder `lvgl:` en niet in het Designer-bestand. Zo gaat hij niet verloren.
3. **Symbolen die alleen het script gebruikt** (sneeuwvlok, bacterie, standby, ontdooien) komen alleen in de lettertypelijst
   als er een label met dat symbool bestaat. Daarom staan er verborgen labels (`overview_icon_cool`, `_standby`, `_leg`,
   `_defrost`). Verwijder ze niet.
4. **Hernoem je een widget**, pas dan ook `ui_scripts.yaml` en `openamber-thermostat.yaml` aan; die verwijzen met id naar de widgets.
5. Staat er `hidden: true` op een container zoals `overview_canvas`, dan verdwijnt alles daarin.

Scripts die iets met widgets doen staan in `ui_scripts.yaml`: `update_status_icons`, `show_pair`, `pair_refresh`,
`pair_confirm`, `refresh_setpoint_ui`, `show_home`, `show_overview` en `knob_step`.

## Flashen

- **Eerste keer of bij een kapotte bootloader**: via USB, met `firmware.factory.bin` (bijvoorbeeld via web.esphome.io).
  Het `ota`-bestand heeft geen bootloader en maakt het apparaat onbruikbaar als je het op die manier flasht.
- **Daarna**: draadloos, via ESPHome Builder (Install, Wirelessly). Dat compileert en uploadt in één keer.
  Een kant-en-klaar `firmware.ota.bin` kun je uploaden met `esphome upload <bestand.yaml> --device <ip>`; die compileert niet.
- **Volgorde bij wijzigingen aan beide kanten**: eerst OpenAmber, dan de thermostaat.
- De `web_server` staat uit; die gebruikt veel geheugen.

## Instellingen die je kunt aanpassen

Bovenin `openamber-thermostat.yaml` (en `led_bodem` in het bordbestand), onder `substitutions:`:

| Naam | Standaard | Betekenis |
|---|---|---|
| `led_bodem` (bordbestand) | 8.5 | positie onderaan de ring (tussen led 8 en 9) |
| `vat_leeg` | 20 | tapwatertemperatuur waarbij het vat als leeg getoond wordt |
| `vat_doel_standaard` | 55 | doel als OpenAmber geen tapwater-setpoint doorgeeft |
| `condens_marge` | 2 | waarschuwen als de aanvoer minder dan zoveel graden boven het dauwpunt komt (bij koelen) |

**Condens**: alleen bij koelen, als de aanvoertemperatuur (Tuo) minder dan `condens_marge` graden boven het dauwpunt van de
kamer (gemeten door de BME680) komt. De uitkomst staat in de globale `condens_warn`.

**Verbinding**: de thermostaat telt als verbonden zolang er in de afgelopen 60 seconden een pakket van de gekoppelde OpenAmber kwam.

## Bekende problemen en valkuilen

- **Zwart of grijs scherm na ESPHome 2026.10.1.** Op die versie bleef het scherm leeg (ook met het oorspronkelijke bestand),
  terwijl 2026.9.1 het goed deed. De oorzaak is niet gevonden; er is geen melding in de changelog over `mipi_rgb`.
  Bouw het project daarom met 2026.9.1, of test een nieuwere versie eerst met een minimaal bestand dat alleen een kleurvlak toont.
- **Eerst `factory`, dan `ota`**: zie Flashen.
- **Namen kloppen niet**: bij "Unknown key"-meldingen eerst controleren of de provider-naam gelijk is aan de `esphome: name:`
  van het andere apparaat, en of `rolling_code_enable` uit staat.
- **Geen data na het flashen**: controleer of beide kanten dezelfde `thermostat_link_key` hebben, de koppelcode klopt, en op OpenAmber
  de thermostaatkeuze op Intern staat.
- **Pakketgrootte**: er worden veel waarden verstuurd (ESP-NOW heeft een beperkte pakketgrootte). Controleer na het flashen in de
  log of er geen meldingen over te grote pakketten zijn; zo wel, haal dan waarden weg.
- **Het pad in `packages:`** is relatief aan de map van het hoofdbestand. "No such file or directory" betekent bijna altijd
  een verkeerd pad of een verkeerde bestandsnaam.
- **Logniveau**: zet `espnow` niet op `DEBUG` of `VERBOSE`; dat geeft te veel logregels en kost geheugen. `lvgl` alleen aan als je een schermprobleem zoekt.
