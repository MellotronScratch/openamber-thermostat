# Backlog

Verwijs naar een item met zijn nummer (bijv. B03). Nieuwe items krijgen het volgende vrije nummer; nummers worden niet
hergebruikt. Een afgerond item wordt doorgestreept en blijft staan.

## Scherm en bediening

- **B01** Glitches bij draaien aan de knop. Waarschijnlijk door de ledring (B17); kijken of ze met de ring uit nog voorkomen.
  Anders: `buffer_size` in het bordbestand van 25% naar 100%.
- **B02** Koppelpagina eventueel als carrousel (zoals menu en schema) in plaats van de roller; eerst de opgemaakte roller
  bekijken (`text_line_space` is geschat).
- **B03** Thema wisselen: controleren of de achtergrond meteen van kleur verandert of pas na een herstart.
- **B04** Schema: na "Bewaard" of kopiëren wordt het setpoint meteen opnieuw uit het schema gezet; nog beslissen of dat zo moet.
- **B05** Schema: ‹ › in de blauwe pil bij wijzigen weggelaten (onzeker of Quicksand die tekens heeft).
- **B06** Schema-titel wijkt af van de andere pagina's (kleiner, grijs `0x8E94A3`), zodat de bovenste rij er niet onder valt.
- **B07** Kamertemperatuurmarker op de ring, instelbare gloedkleur, verloop in de bezel, contrast van tekstkleuren.
- **B08** Een tweede scherm (Nextion NX4848E021-011C).
- **B22** Instelling "Groot in beeld": doeltemperatuur of kamertemperatuur groot in het midden (de andere klein eronder).
  Idee: bij draaien tijdelijk de doeltemperatuur groot, na een paar seconden terug; bij de kleine temperatuur "Doel" of
  "Kamer" zetten. De ring blijft de doeltemperatuur tonen.
- **B23** QR-code naar de repository (README) op het scherm. De pagina Info bestaat inmiddels (IP-adres, MAC-adres,
  WiFi-signaal), maar zonder QR-code. Waar de QR-code komt, nog bedenken (bijvoorbeeld op Info of op Software-update).

## Designer

- **B09** `schema_canvas`, `fout_canvas` en `inst_canvas` los in de Designer plakken, anders verdwijnen ze bij de volgende export.
- **B10** In de Designer `*_bezel_1` los op de pagina zetten (x 0, y 0) in plaats van in de achtergrond, en de opmaak van de
  koppelpagina overnemen.
- **B11** Nakijken of de Designer `opa` (dekking van de schema-rijen) bewaart.

## Firmware

- **B12** Twee `-Wformat-truncation`-waarschuwingen in de hulpjes `tijd`/`temp` van `schema_render` (onschuldig; buffer groter maken).
- **B13** IRAM staat op 100%: een nieuw component dat IRAM wil, kan het compileren laten mislukken.
- **B14** Pakketgrootte van ESP-NOW: tot nu toe geen fouten gezien in de logs van OpenAmber; blijven letten op meldingen
  over te grote pakketten.
- ~~**B15** Foutcodes: nu alleen de eerste actieve code (`ec`). Alle codes kan later met twee bitmasker-kanalen vanuit OpenAmber.~~
  Klaar: OpenAmber stuurt de vlaggen als bits (`e1`, `e2`); de pagina Foutcodes toont alle actieve codes.
- **B16** Het effect "Vat" wordt niet meer gebruikt; eventueel opruimen (ook `vat_leeg`, `vat_doel_standaard` en `led_bodem`).
- **B17** Ledring: het effect "Status" verstuurt de leds elke 40 ms (`schedule_show()` bij elke update, ook als de kleur
  gelijk blijft). Met de ring aan: glitches op het scherm en de thermostaat valt van de WiFi; met de ring uit 15 minuten
  stabiel. Daarom staat `ledring_beschikbaar` op `"false"`. Later: alleen versturen bij een kleurwijziging en een trager
  interval, daarna weer beschikbaar maken. Waarschijnlijk ook de oorzaak van B01.
  Let op: daarna viel de thermostaat ook met de ring uit nog een paar keer van de WiFi (niet bereikbaar voor OTA en ping).
  De ring is dus waarschijnlijk niet de enige oorzaak; nog uitzoeken (bijvoorbeeld een log over langere tijd).
- **B21** De thermostaat valt geregeld van het netwerk (geen ping, geen OTA, niet via mDNS) en komt pas terug na
  stroomloos maken. Ook gezien bij OTA-uploads die halverwege afbreken. Mogelijk dezelfde oorzaak als B17; uitzoeken met
  een log over langere tijd (bijvoorbeeld via USB).
- **B24** Overstappen van ESPHome 2026.9.1 naar 2026.11.0 zodra die uit is. 2026.10.x geeft een zwart scherm
  ([esphome#21785](https://github.com/esphome/esphome/issues/21785), opgelost in 2026.11.0). Eerst via USB testen, met de
  vorige firmware bij de hand om terug te gaan. Daarna ook de release-workflow en de vermelding van 2026.9.1 in de
  README en CLAUDE.md bijwerken.
- **B19** Bij "Geen verbinding" blijven de waarden van OpenAmber (temperaturen, setpoint) staan met hun laatste waarde;
  liever leeg tonen ("--") zolang er geen verbinding is.

## Software-update

- **B18** Software-update via GitHub: release-workflow met manifest en de update-pagina met het update-component
  (`http_request`) zijn er (0.0.2). Nog testen: updaten naar 0.0.3 via de pagina.
- **B20** Koppelpagina: ben je al gekoppeld, dan eerst de huidige koppeling tonen (koppelcode en gekoppelde OpenAmber),
  met de keuze om opnieuw te koppelen met een nieuwe code. Nu kom je meteen in het invoeren van een code. Voor 0.0.3.
