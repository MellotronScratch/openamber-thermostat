# OpenAmber Thermostaat

Een rond draaiknop-display (ESP32-S3, ESPHome 2026.9.1, LVGL 9) dat als thermostaat in de woonkamer
hangt en via ESP-NOW praat met het OpenAmber-display op het binnendeel van de warmtepomp.
Gesproken en geschreven wordt in het Nederlands.

## Hoe we werken (belangrijk)
- Doe alleen wat gevraagd wordt, in het tempo van Felix. Vraagt hij of je ergens mee uit de voeten kunt,
  of vraagt hij alleen om de LVGL, maak dan geen scripts of bestanden aan. Implementeer pas als hij zegt dat hij zover is.
- Houd oplossingen eenvoudig. Geen splitsingen of hulpscripts die hij zelf moet draaien.
- Vraag eerst voordat je nieuwe bestanden aanmaakt.
- Felix is niet de ontwikkelaar van OpenAmber. Deze repo bevat alleen de thermostaat. De OpenAmber-kant
  (`src/openamber/thermostat_link/thermostat_link.yaml`) staat in de OpenAmber-repo (fork MellotronScratch/openamber,
  branch `feature/ESPNOW`, PR naar Jordi1990/openamber); daar wordt alleen in overleg iets aan veranderd.
- De firmware wordt kant-en-klaar aan eindgebruikers geleverd (geen compileren, alleen updaten via de GUI).
  Andere mensen gaan dit project gebruiken, dus geen aannames over Home Assistant of MQTT.

## Hardware
- Scherm/bord: ZX2D10GE01R-V4848 (WT32S3-21S), 480x480 rond, 13 WS2812-leds, alleen draaiknop en druktoets, geen aanraking.
- Sensor: BME680 op I2C (SCL GPIO1, SDA GPIO2, adres 0x77).
- Bordspecifieke code staat in `openamber-thermostat-ZX2D10GE01R-V4848.yaml`; de generieke code in `openamber-thermostat.yaml`.
  Een ander scherm krijgt een eigen bordbestand. Het VIEWE-bordbestand is geparkeerd en verouderd.

## Koppeling met OpenAmber (ESP-NOW, packet_transport)
- Broadcast met gedeelde sleutel: substitution `thermostat_link_key` (vaste waarde `openamber-thermostat-link`, gelijk aan de standaard in OpenAmber). Koppelen gaat met de laatste 6 hexadecimale tekens van het MAC-adres van OpenAmber
  (de koppelcode). Die kiest de gebruiker op de thermostaat uit een lijst.
- OpenAmber naar thermostaat (sensors): pin, tw, tws, tc, tuo (aanvoer), tui (retour), ta, rt, slo, shi, mode, cf (Hz),
  ec (foutcode: 1xx=P, 2xx=F, 3xx=E, 4xx overig, 0=geen), wsp (huidig setpoint = gewenste aanvoer), tm (tijd, minuten sinds maandag 00:00).
  Binair: err, warn, wv, cd, twp, twv, da, lega, dfr, bu, twa, legp.
- Thermostaat naar OpenAmber: pin, bt, bh, dp, cmd.
- De pakketgrootte met alle kanalen is nog niet getest. Let in de logs op een fout daarover.

## Bestanden (hoofdmap)
- `openamber-thermostat.yaml`: generiek: WiFi/API, packet_transport-sensors, setpoint-nummers, modus, klok, knop, scripts.
- `thermostat/ui_designer.yaml`: ALLEEN door de LVGL Designer (https://lvgl.espboards.dev) gegenereerd. Niet met de hand wijzigen,
  Felix plakt hier zijn export. Eén bestand, ook al importeert de Designer bij plakken maar de eerste pagina; we splitsen het bewust niet.
- `thermostat/ui_scripts.yaml`: scripts die de pagina's bedienen (menu, statusiconen, modus, koppelen, overzicht, knop).
  Verwijst naar widget-id's uit de Designer; hernoemt hij er een, pas het hier aan.
- `thermostat/ui_fonts.yaml`: eigen icoonfonts (MDI) voor wat het script nodig heeft; ze worden bij het opstarten op de widgets gezet.
- `thermostat/ui_tikken.yaml`: vulling en kleurverloop van de tikken rond het setpoint (eigen callback op de meter).
- `thermostat/schema.yaml`: nachtverlaging per weekdag (entiteiten, tijdbron, periodiek toepassen).
- `openamber-thermostat-emulator.yaml`: emulator op de laptop (ESPHome `host` + SDL-venster); start met `esphome run openamber-thermostat-emulator.yaml`.
  Pijltje links/rechts = draaien, Enter = korte druk, Backspace = lange druk. Laadt het generieke deel.
- `thermostat/emulator_weg.yaml`: haalt voor de emulator weg wat niet op een laptop kan (`!remove`: wifi, ota, espnow, ESP-NOW-koppeling, sntp).

## Pagina's en bediening (knob_context)
0 Temperatuur (`climate_canvas`, setpoint instellen), 1 Menu (`carrousel_canvas`, carrousel met loop), 2 Modus (`modus_canvas`),
3 Koppelen (`pair_canvas`), 4 Overzicht (`overzicht_canvas`), 5 Schema (`schema_canvas`, overzicht, dag en wijzigen; lange druk gaat daar een stap terug),
6 Foutcodes (`fout_canvas`, eerste actieve code van `ec` met de tekst uit OpenAmber), 7 Instellingen (`inst_canvas`, ledring, helderheid, thema, sensor: welke kamertemperatuur het scherm toont, Tr of BME680; OpenAmber regelt altijd op Tr).
Draaien = kiezen, korte druk = openen of bevestigen, lange druk = terug. Zonder bediening blijft de pagina staan, ook als het scherm
uit is; na 5 minuten (`pagina_terug_min`) gaat hij terug naar Temperatuur en staat het menu weer op Temperatuur.
Het menu is 7 items (Temperatuur, Overzicht, Schema, Foutcodes, Modus, Koppelen, Instellingen).
Ledring: effect "Status" (bordbestand), aangestuurd door `led_update`. Scherm aan = kleur van de modus, ademt als de compressor draait,
korte flits bij bediening; scherm uit = alleen fout (rood) of waarschuwing (oranje). Het effect "Vat" bestaat nog maar wordt niet gebruikt.
Thema (select `thema`): achtergrondverloop (`achtergrond_verloop`, diagonaal, bij het opstarten op alle achtergronden gezet) en accentkleur samen.

## Technische afspraken
- Het Designer-blok wordt bij elke export opnieuw gegenereerd; handmatige wijzigingen daarbinnen gaan verloren.
  Wijzigingen daarbuiten blijven. Onbekende eigenschappen worden weggegooid.
- Widget-id's zijn in lambdas `lv_obj_t*` (bar, label, obj); een roller heeft een wrapper (`->obj`).
- Pakketten voegen lijsten samen op id (fonts, globals, scripts, `esphome: on_boot`).
- Substituties worden ook in lambdas uitgebreid.
- Het scherm gaat na 60 s uit; de eerste druk of draai maakt het alleen wakker.
- Alleen het verwarmingssetpoint wordt door het schema gestuurd, en alleen in de modus Verwarmen.

## Stand van zaken
- Klaar maar niet op de hardware getest: statusiconen, overzichtskleuren (stooklijn via `wsp`), pagina-geheugen van 5 minuten, tikken rond het setpoint, schema-entiteiten, tijd via ESP-NOW.
- Schema op het scherm: `schema_canvas` staat in `ui_designer.yaml` (rijen `schema_rij_1`-`_5`, rij 3 is de geselecteerde; onderdelen `_naam`, `_bar`, `_dagtemp`, `_nachttemp`, `_waarde`, plus `schema_title`, `schema_hint`, `schema_toast`). Scripts staan in `ui_scripts.yaml` (`schema_render`, `_draai`, `_druk`, `_terug`, `_melding`); nog niet op de hardware getest.
- OpenAmber moet een build met `thermostat_link.yaml` hebben (voorlopig alleen het Waveshare-display).
- `esphome compile` werkt lokaal (hoofdmap, bordbestand); compileer na C++-wijzigingen in lambdas, `esphome config` controleert die niet.

## Backlog
Verwijs naar een item met zijn nummer (bijv. B03). Nieuwe items krijgen het volgende vrije nummer; nummers worden niet hergebruikt.
Scherm en bediening
- B01 Glitches bij draaien aan de knop. Proef: `buffer_size` in het bordbestand van 25% naar 100%, en kijken of het met de ledring uit ook gebeurt.
- B02 Koppelpagina eventueel als carrousel (zoals menu en schema) in plaats van de roller; eerst de opgemaakte roller bekijken (`text_line_space` is geschat).
- B03 Thema wisselen: controleren of de achtergrond meteen van kleur verandert of pas na een herstart.
- B04 Schema: na "Bewaard" of kopiëren wordt het setpoint meteen opnieuw uit het schema gezet; Felix moet nog zeggen of hij dat wil.
- B05 Schema: ‹ › in de blauwe pil bij wijzigen weggelaten (onzeker of Quicksand die tekens heeft).
- B06 Schema-titel wijkt af van de andere pagina's (kleiner, grijs `0x8E94A3`), zodat de bovenste rij er niet onder valt.
- B07 Kamertemperatuurmarker op de ring, instelbare gloedkleur, verloop in de bezel, contrast van tekstkleuren.
- B08 Een tweede scherm (Nextion NX4848E021-011C).

Designer
- B09 `schema_canvas`, `fout_canvas` en `inst_canvas` los in de Designer plakken, anders verdwijnen ze bij de volgende export.
- B10 In de Designer `*_bezel_1` los op de pagina zetten (x 0, y 0) in plaats van in de achtergrond, en de opmaak van de koppelpagina overnemen.
- B11 Nakijken of de Designer `opa` (dekking van de schema-rijen) bewaart.

Firmware
- B12 Twee `-Wformat-truncation`-waarschuwingen in de hulpjes `tijd`/`temp` van `schema_render` (onschuldig; buffer groter maken).
- B13 IRAM staat op 100%: een nieuw component dat IRAM wil, kan het compileren laten mislukken.
- B14 Na het flashen in de logs letten op een fout over de pakketgrootte van ESP-NOW.
- B15 Foutcodes: nu alleen de eerste actieve code (`ec`). Alle codes kan later met twee bitmasker-kanalen vanuit OpenAmber.
- B16 Het effect "Vat" wordt niet meer gebruikt; eventueel opruimen.
- B17 Thermostaat valt van de WiFi zodra hij gekoppeld is; vermoedelijk door het effect "Status" (40 ms, ademt als de compressor draait). Ledring staat daarom standaard uit. Proef: trager effect (100-200 ms) of zonder ademen. Ook nog: debug-logging (espnow, packet_transport) terug naar INFO.
