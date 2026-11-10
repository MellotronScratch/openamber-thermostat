# Werking van de koppeling

Hoe de thermostaat en OpenAmber met elkaar praten, en welk bestand wat doet. De bediening op het scherm staat in de
[README](../README.md).

## De bestanden

**OpenAmber** ([PR #305](https://github.com/Jordi1990/openamber/pull/305), branch `dev`)

| Bestand | Wat |
|---|---|
| `src/openamber/thermostat_link/thermostat_link.yaml` | het pakket met alles van de koppeling (hieronder "het pakket") |
| `src/openamber-waveshare-display.yaml` | neemt het pakket op (één regel) |
| `src/openamber/ui/settings/page_settings_tab_thermostat.yaml` | Thermostaat → Algemeen in twee pagina's; pagina 2 heeft de rijen "Draadloze thermostaat" (schakelaar) en "Koppelcode draadloze thermostaat" |
| `src/openamber/ui/page_settings.yaml` | de paginabalk ‹ Pagina 1 / 2 › van dat tabblad (`thermostat_algemeen_pager`) |
| `src/openamber/ui/bindings/script_ui_bindings.yaml` | pagina wisselen (`thermostat_algemeen_select_page`), "Zeker?" bij Nieuw (`thermostat_link_new_code_confirm`) en de lege scripts `thermostat_link_toggle_ui` en `thermostat_link_new_code_ui` |

**Thermostaat** (deze repo)

| Bestand | Wat |
|---|---|
| `openamber-thermostat.yaml` | ESP-NOW, ontvangen en versturen, koppelcode en sleutel, setpoints, modus en commando's |
| `thermostat/ui_scripts.yaml` | de pagina's, waaronder de koppelpagina |
| `thermostat/schema.yaml` | weekschema; zet het verwarmingssetpoint via een commando |
| `openamber-thermostat-ZX2D10GE01R-V4848.yaml` | bordbestand (scherm, knop, BME680, ledring) |

De overige bestanden (`ui_designer.yaml`, `ui_fonts.yaml`, `ui_tikken.yaml`, emulator) hebben niets met de koppeling te maken.

## De verbinding

Beide kanten gebruiken het ESPHome-onderdeel `packet_transport` over ESP-NOW:

- **Broadcast**: elk apparaat roept zijn waarden naar iedereen in de buurt; er staan geen MAC-adressen in de code.
  ESP-NOW gebruikt het WiFi-kanaal, dus beide apparaten moeten op hetzelfde kanaal zitten.
- **Namen**: elke waarde gaat over onder een korte naam (`broadcast_id`, bijvoorbeeld `tw`), en de ontvanger vraagt die
  naam op bij een bepaald apparaat (`provider` + `remote_id`). Het apparaat herkent de afzender aan zijn
  `esphome: name:` (`openamber` en `openamber-thermostat`).
- **Versleuteld**: elk pakket is versleuteld met dezelfde sleutel aan beide kanten (zie Koppelen).
- **Wanneer**: een waarde wordt verstuurd zodra hij verandert, en daarnaast elke 15 s opnieuw.
- **Alleen getallen en aan/uit-waarden**. Een climate (modus plus setpoints) kan niet over, daarom leest het pakket de
  setpoints en de modus uit in gewone sensoren.

Welke waarden er over en weer gaan, staat in de README onder "Overzicht van alle waarden".

## Hoe een bericht eruitziet

Een bericht is geen leesbare tekst maar een rij bytes. Alleen de kop (wie het stuurt) is leesbaar; de inhoud is
versleuteld met de sleutel uit de koppelcode (XXTEA) en ziet er in de lucht uit als willekeurige bytes. Hieronder staat
een bericht zoals het eruitziet vóór het versleutelen (opbouw volgens `packet_transport` in ESPHome).

**Van de thermostaat naar OpenAmber** (68 bytes):

```
--- kop, niet versleuteld ---
53 45                           vast herkenningsteken ("ES" = ESPHome)
14                              lengte van de naam: 20 tekens
6f 70 65 6e 61 6d 62 65 72 2d   "openamber-"
74 68 65 72 6d 6f 73 74 61 74   "thermostat"  (de afzender, esphome: name:)
00                              opvulling tot een veelvoud van 4 bytes

--- inhoud, versleuteld ---
01                              begin van de gegevens
02 23 c1 2b 4b 03 70 69 6e      sensor 11256099 (= hex ABC123)  "pin"  MAC-code van de gekoppelde OpenAmber
02 66 66 aa 41 02 62 74         sensor 21,3                     "bt"   temperatuur BME680
02 00 00 40 42 02 62 68         sensor 48,0                     "bh"   luchtvochtigheid
02 00 00 20 41 02 64 70         sensor 10,0                     "dp"   dauwpunt
02 80 21 51 48 03 63 6d 64      sensor 214150                   "cmd"  commando
00                              opvulling tot een veelvoud van 4 bytes
```

**Van OpenAmber naar de thermostaat** (een deel; het echte bericht bevat alle waarden):

```
53 45 09 6f 70 65 6e 61 6d 62 65 72      kop: naam "openamber" (9 tekens)
01                                       begin van de gegevens
02 23 c1 2b 4b 03 70 69 6e               sensor 11256099 (= hex ABC123)  "pin"  eigen MAC-code
02 00 00 42 42 02 74 77                  sensor 48,5                     "tw"   tapwater
02 00 00 28 42 02 63 66                  sensor 42,0                     "cf"   compressor (Hz)
03 00 03 65 72 72                        aan/uit: uit                    "err"  geen fout
03 01 02 77 76                           aan/uit: aan                    "wv"   warmtevraag
```

Elke waarde bestaat uit:

| Deel | Lengte | Betekenis |
|---|---|---|
| soort | 1 byte | `02` = sensor (getal), `03` = aan/uit-waarde |
| waarde | 4 bytes of 1 byte | sensor: kommagetal (float), minst significante byte eerst; aan/uit: `00` of `01` |
| lengte naam | 1 byte | aantal tekens van de naam |
| naam | zoveel bytes | de naam (`broadcast_id`), bijvoorbeeld `tw` |

Het commando 214150 pakt OpenAmber uit volgens de formule in de README (Commando's van thermostaat naar OpenAmber):
volgnummer 5, modus 1 (verwarmen), laag 41 / 2 = 20,5°, hoog 50 / 2 = 25°.

Wat je hieruit kunt halen:

- **Korte namen** (`tw`, `cf`): elke naam gaat in elk bericht mee, dus korte namen houden het bericht klein.
- **Alleen getallen**: er is geen soort voor tekst, daarom gaat de MAC-code als getal over. Een float is precies tot
  24 bits, genoeg voor 6 hex-tekens. De koppelcode zelf gaat nooit over: die zit alleen in de sleutel.
- **Grootte**: past niet alles in één ESP-NOW-bericht, dan begint `packet_transport` zelf een volgend bericht
  (met dezelfde kop). Een te groot bericht hoort dus niet voor te komen.

## Opstarten

**OpenAmber**: de koppeling staat standaard uit. ESP-NOW start uitgeschakeld (`enable_on_boot: false`) en gaat pas aan
met de schakelaar "Draadloze thermostaat" (`thermostat_link_enabled`), bij het opstarten (`on_boot`) of zodra je hem
aanzet. De schakelaar op het scherm roept `thermostat_link_toggle_ui` aan, dat het pakket uitbreidt met het omzetten van
de schakelaar. Uitzetten zet ESP-NOW uit, `thermostat_pin_ok` op onwaar en de BME680-waarden op "onbekend". De triggers
van de schakelaar doen pas iets als ESP-NOW is opgestart: bij het terugzetten van de bewaarde stand tijdens het opstarten
zou ESP-NOW anders crashen. Met de schakelaar aan doet `thermostat_link_apply_code`:
1. Is er nog geen koppelcode (`thermostat_link_code` is 0), dan maakt hij een willekeurige code van 6 cijfers
   (100000-999999, met `random_bytes`).
2. Hij berekent de sleutel: SHA-256 van `thermostat_link_key` + ":" + de code, en zet die als eigen sleutel en als
   sleutel voor de thermostaat (`set_encryption_key`, `set_provider_encryption`).
3. Hij toont de code in de tekstsensor "Thermostaat koppelcode" en op het scherm (`thermostat_link_ui`: de rij met de
   schakelaar is altijd zichtbaar, de rij met de code alleen met de schakelaar aan).
4. Na 2 s schrijft hij de instellingen naar flash, zodat een nieuwe code niet verloren gaat bij een crash.

Bij het opstarten schrijven beide kanten ook meteen de teller van de rolling code naar flash (zie Commando's).
De entiteiten van de koppeling staan in Home Assistant standaard uit (`disabled_by_default`).

Tot stap 2 geldt de vaste `thermostat_link_key` als sleutel. Zo wordt er nooit onversleuteld verstuurd of ontvangen;
`packet_transport` zou zonder sleutel ook onversleutelde pakketten aannemen.

**Thermostaat** (`link_key_apply`): heeft hij een opgeslagen koppelcode (`link_code`), dan berekent hij op dezelfde
manier dezelfde sleutel. Zonder code blijft de vaste sleutel staan en hoort hij geen enkele OpenAmber; na 3 s opent
dan vanzelf de koppelpagina.

## Koppelen

1. Op de thermostaat voer je de code in op de koppelpagina (`show_pair`, `pair_druk`): de roller kiest een cijfer, een
   druk legt het vast. Een code die met 0 begint, kan niet en wordt gewist.
2. Na het zesde cijfer bewaart de thermostaat de code (`link_code`), maakt `pair_code` leeg en zet de nieuwe sleutel.
3. Elk pakket van OpenAmber bevat `pin`, de **MAC-code**: de laatste 6 hex-tekens van het MAC-adres van OpenAmber. Dat
   is niet de koppelcode; de koppelcode maakt de sleutel, de MAC-code zegt welke OpenAmber het is. Alleen een OpenAmber
   met dezelfde koppelcode komt door de versleuteling heen.
4. **Eerste wint**: het eerste pakket dat binnenkomt, legt die MAC-code vast in `pair_code` (`remote_pin`). Daarna
   gebruikt de thermostaat alleen waarden met precies die `pin` (`pin_ok`), dus ook bij twee OpenAmbers met dezelfde code.
5. De koppelpagina (`pair_refresh`, elke 2 s) toont "Verbinden met …", dan "Gekoppeld met OpenAmber 1A2B3C", of na
   30 s "Geen OpenAmber gevonden".
6. **Andersom**: de thermostaat stuurt `pair_code` terug als zijn eigen `pin` (`pin_out`). OpenAmber vergelijkt die met
   zijn eigen MAC-code (`thermostat_pin_check`) en neemt alleen dan BME680-waarden en commando's aan (`thermostat_pin_ok`).

## Waarden van OpenAmber naar de thermostaat

- Temperaturen en statussen gaan als gewone sensoren en binary sensors over.
- **Setpoints en modus**: `thermostat_link_target_low`, `_target_high` en `_mode` lezen de climate elke 5 s uit. De
  modus gaat over als ESPHome-getal (`climate::ClimateMode`: 0 uit, 2 koelen, 3 verwarmen).
- **Foutcodes** (`thermostat_link_errors_1` en `_2`, kanalen `e1` en `e2`): elke fout- en waarschuwingsvlag van
  OpenAmber is één bit. De thermostaat maakt er een lijst met actieve codes van (`fout_bijwerken`, `fout_codes`), in
  volgorde van belang; de statusregel toont de eerste, de pagina Foutcodes alle. De bitvolgorde staat in de README.
- **Tijd** (`thermostat_link_time`): minuten sinds maandag 00:00, voor het schema van de thermostaat.
- Op de thermostaat filtert elke ontvangen sensor op `pin_ok`; zonder koppeling worden de waarden "onbekend".
  `last_rx` houdt bij wanneer het laatste pakket kwam: na 60 s zonder pakket toont de thermostaat "Geen verbinding".

## Commando's van de thermostaat naar OpenAmber

1. Je draait aan de knop of kiest een modus. De thermostaat wacht 600 ms na de laatste klik (`send_command`) en
   verstuurt dan één getal (`cmd`):
   `((volgnummer * 4 + modus) * 100 + laag * 2) * 100 + hoog * 2`, met modus 3 = "ongewijzigd laten".
2. Het volgnummer loopt van 1 tot 99 en blijft bewaard na een herstart (`cmd_seq`).
   Daarnaast heeft elk pakket de teller van de rolling code: OpenAmber weigert een pakket met een oudere teller, zodat
   een opgevangen commando niet opnieuw kan worden uitgezonden.
3. OpenAmber pakt het getal uit (`thermostat_rx_cmd`):
   - alleen als `thermostat_pin_ok` waar is;
   - een volgnummer dat gelijk is aan het vorige (`thermostat_last_cmd_seq`) wordt genegeerd, want het pakket wordt
     elke 15 s herhaald;
   - de modus wordt gezet, en alleen het setpoint van de actieve modus: bij koelen het hoge, anders het lage (5-35 °C);
   - daarna leest hij de setpoints en de modus meteen opnieuw uit, zodat de thermostaat niet 5 s de oude waarde terugkrijgt,
     en werkt hij zijn scherm bij (`update_thermostat_settings_ui`).
4. Zolang je draait (en 3 s erna) negeert de thermostaat de setpoints en de modus die OpenAmber terugstuurt
   (`user_is_adjusting`), anders springt het scherm heen en weer.

## Nieuwe koppelcode

- Op het scherm van OpenAmber: "Nieuw" wordt bij de eerste tik "Zeker?" (rood). Een tweede tik binnen 5 s roept
  `thermostat_link_new_code_ui` aan; het pakket breidt dat lege script uit met een druk op de knop
  "Thermostaat nieuwe koppelcode". Zonder pakket doet de knop niets.
- De knop zet de code op 0 en start `thermostat_link_apply_code`: een nieuwe code, een nieuwe sleutel,
  `thermostat_pin_ok` uit en de BME680-waarden van de oude thermostaat op "onbekend".
- De thermostaat hoort OpenAmber daarna niet meer en moet opnieuw gekoppeld worden.

## Beperkingen

- Een code van 6 cijfers is met wat rekenwerk te raden uit opgevangen pakketten.
- Herstart OpenAmber, dan neemt hij het eerste pakket van de thermostaat aan zonder de teller te kunnen vergelijken
  (de laatst geziene teller staat alleen in het geheugen). Een opgevangen pakket werkt dus hoogstens één keer, direct
  na een herstart.
- Een OpenAmber waarvan het MAC-adres eindigt op 00:00:00 kan niet koppelen (0 betekent "nog geen code").
