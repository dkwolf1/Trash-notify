# Trash Notify

[English](README.md) · [Nederlands](README.nl.md)

**Trash Notify** is een Home Assistant-blueprint voor afvalherinneringen met optionele **Ja/Nee-knoppen** op de telefoon en instelbare herhalingen.

Een herstart van Home Assistant Core of het herladen van automatiseringen wist een actieve herinnering niet. De volgende herinneringstijd, ophaaldatum, het aantal verzonden meldingen en de bevestiging staan in een blijvende `input_text`-helper. Een controle per minuut hervat de cyclus na het opstarten. Extra afval- of templatesensoren, AppDaemon en aangepaste integraties zijn niet nodig.

> **Versie:** 0.2.3 — testversie voor Home Assistant OS 2026.10 of nieuwer. Eén cyclus met Nee en daarna Ja is op een echte installatie waargenomen. Herstel na een Core-herstart en verzending naar twee toestellen moeten nog in de praktijk worden gecontroleerd voordat deze versie stabiel genoemd kan worden.
>
> **Belangrijk:** voor Ja/Nee is **één `input_text`-helper per Trash Notify-automatisering** nodig. Voor een eenmalige melding zonder bevestiging is geen helper nodig.

## Blueprint

[Importeer de blueprint in Home Assistant](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fdkwolf1%2FTrash-notify%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Ftrash_notify.yaml) · [Bekijk de YAML](blueprints/automation/trash_notify.yaml)

## Functies

- Ondersteunt maximaal vier afvalsensoren. Hun **status moet exact gelijk zijn aan de ophaaldatum**; het standaardformaat is `%d-%m-%Y`.
- Je stelt het eerste meldingstijdstip en het aantal dagen vóór ophalen in (`1` betekent morgen). Het datumformaat geldt voor de sensorstatus. Een expliciet opgeslagen datumformaat in een bestaande automatisering verandert niet vanzelf.
- Met de optionele knoppen bevestigt **Ja** dat de container buiten staat en wordt de melding gewist. **Nee** plant de volgende herinnering vanaf het moment waarop je drukt.
- Zonder antwoord volgt na het ingestelde interval een nieuwe melding, tot het ingestelde maximum.
- Twee Companion App-telefoons kunnen apart worden ingesteld; de tweede is optioneel. Een notificatiegroep kan ook. **Ja op één telefoon** bevestigt de gezamenlijke cyclus en wist de melding op beide toestellen.
- Herhaalde meldingen vervangen de vorige melding via een vaste notificatietag.
- Een lopende cyclus wordt na een Core-herstart of het herladen van de automatiseringen hervat. Als Core op het oorspronkelijke tijdstip uit stond, kan de cyclus later op **dezelfde kalenderdag** nog beginnen.
- Antwoorden op meldingen van een andere ophaaldatum worden genegeerd.

## Installatie

1. Gebruik de importlink hierboven of ga naar **Instellingen → Automatiseringen & scènes → Blueprints → Blueprint importeren** en plak de [blueprint-URL](https://github.com/dkwolf1/Trash-notify/blob/main/blueprints/automation/trash_notify.yaml). Handmatig kun je het YAML-bestand onder `/config/blueprints/automation/dkwolf1/trash_notify.yaml` plaatsen.
2. Maak voor **Ja/Nee** een Tekst-helper via **Instellingen → Apparaten & diensten → Helpers → Helper maken → Tekst**:
   - Naam: `Trash Notify State`
   - Voorgestelde entiteits-ID: `input_text.trash_notify_state`
   - **Minimumlengte: 0 tekens**
   - **Maximumlengte: 255 tekens**. De standaardlimiet van 100 is te kort voor de opgeslagen JSON.
   - Laat een validatiepatroon leeg.
   - Gebruik bij YAML-configuratie geen `initial`-waarde, zodat de vorige status na een herstart wordt hersteld.
3. Je kunt de helper ook in `configuration.yaml` zetten:

   ```yaml
   input_text:
     trash_notify_state:
       name: Trash Notify State
       min: 0
       max: 255
       # Belangrijk: geen initial
   ```

   Voeg dit toe aan een bestaand `input_text:`-blok als dat er al is. Herlaad daarna de Input Text-integratie of herstart Home Assistant.
4. Importeer de blueprint of maak er een automatisering van. Kies de afvalsensoren, het eerste meldingstijdstip en **Notificatieapparaat 1** (`notify.mobile_app_*`). Stel eventueel een andere service in bij **Notificatieapparaat 2**.
5. Schakel **Vraag om Ja/Nee-bevestiging** in, kies de helper bij **Statusopslag voor bevestigingen** en stel het herhaalinterval en het maximum aantal meldingen in.
6. Schakel een oude afvalherinnering uit om dubbele meldingen te voorkomen.

Importeer de blueprint na wijzigingen in deze repository opnieuw in Home Assistant en herlaad de automatiseringen. Controleer bij bestaande automatiseringen het veld **Datumformaat van de sensoren**: een expliciet opgeslagen oude waarde moet je zelf aanpassen.

Gebruik voor meerdere Trash Notify-automatiseringen **een aparte `input_text`-helper per automatisering**. Anders kunnen de opgeslagen status en knop-ID’s elkaar beïnvloeden.

## Hoe herinneringen na een herstart werken

De helper bevat compacte JSON, bijvoorbeeld:

```json
{"day":"2026-10-18","due":"2026-10-19","mask":1,"status":"pending","sent":1,"next":1792339200}
```

- `day` is de herinneringsdag, `due` de ophaaldag, `mask` de passende afvalsensoren, `status` is `pending` of `confirmed`, `sent` telt de verzendpogingen en `next` is het Unix-tijdstip van de volgende melding. `day` en `due` blijven intern in ISO-formaat, ook als de sensoren `%d-%m-%Y` gebruiken.
- Na elke `input_text.set_value` schrijft `homeassistant.save_persistent_states` een controlepunt weg.
- De automatisering gebruikt geen lange wachttijd. Iedere minuut en na het opstarten controleert zij of de opgeslagen herinneringstijd is bereikt. Een herstart begint dus geen nieuw volledig interval.
- **Ja** slaat `confirmed` op en wist de melding. Een herstart begint de herinneringen van die dag niet opnieuw.
- **Nee** zet `next` op het antwoordtijdstip plus het ingestelde interval. Zonder antwoord loopt het interval vanaf de laatste verzendpoging.
- Na het maximum worden voor die ophaaldatum geen automatische herhalingen meer verzonden. De Ja-knop van de laatste zichtbare melding kan nog worden gebruikt.

### Voorbeeld

Je ontvangt om 18:00 een melding, drukt om 18:05 op **Nee** en hebt een interval van 60 minuten ingesteld. De volgende melding is gepland rond **19:05**. Herstart Core om 18:30, dan blijft `next` bewaard. Druk je om 18:35 op **Ja**, dan wordt de bevestiging opgeslagen en volgen er voor deze ophaalronde geen meldingen meer.

## Na Ja en bij de volgende ophaalronde

Na **Ja** blijft de helper voor die herinneringsdag op `confirmed`. Alleen het tijdstip van de automatisering wijzigen levert op **dezelfde dag** geen nieuwe melding op. De helper wordt niet automatisch leeggemaakt: de controle per minuut zou anders dezelfde ronde opnieuw kunnen starten.

Op een latere dag overschrijft de blueprint de oude status met een nieuwe `pending`-cyclus **als** het ingestelde tijdstip is bereikt en de sensorstatus overeenkomt met de nieuwe doel-ophaaldatum. Zonder passende sensorstatus kan de oude `confirmed`-status zichtbaar blijven. Die blokkeert een toekomstige passende ophaalronde niet.

Wil je **op dezelfde dag opnieuw testen**, voer dan één keer deze actie uit onder **Ontwikkelaarstools → Acties**:

```yaml
action: input_text.set_value
target:
  entity_id: input_text.trash_notify_state
data:
  value: "{}"
```

Gebruik je eigen entiteits-ID als die anders is. Controleer daarna of de sensorstatus past bij de ingestelde dagafstand en het datumformaat. Als het ingestelde tijdstip al voorbij is, kan de volgende minuutcontrole een nieuwe cyclus starten; anders gebeurt dat op het ingestelde tijdstip.

## Telefoonmeldingen

De Ja/Nee-knoppen vereisen de officiële Home Assistant Companion App op iOS of Android. Gebruik een `notify.mobile_app_*`-service of een geschikte mobiele notificatiegroep. `notify.notify` werkt alleen als de ontvangers acties ondersteunen.

Beide toestellen krijgen dezelfde knop-ID’s en notificatietag. Ja op één toestel slaat één gedeelde bevestiging op en stuurt `clear_notification` naar beide. Nee op één toestel stelt de gedeelde cyclus uit. Ontvangst en wissen hangen ook af van de toestemmingen van het mobiele besturingssysteem en de Companion App.

## Bekende beperkingen

- Eén `input_text`-helper bewaart één actieve ophaalcyclus. Ophaalrondes op opeenvolgende dagen kunnen elkaar overlappen; een nieuwe dagcyclus kan de vorige vervangen. Gebruik aparte automatiseringen en helpers als zulke rondes onafhankelijk moeten blijven.
- Meerdere afvalsoorten op dezelfde ophaaldag krijgen samen één Ja/Nee-beslissing.
- De helper moet beschikbaar zijn, `min: 0`, `max: 255` en geen validatiepatroon hebben. Bij een ontbrekende helper of ongeldige/onvolledige JSON stopt de geplande verwerking met een fout om de opgeslagen cyclus niet stilzwijgend te vervangen.
- De controle loopt per minuut. Een melding kan na een herstart of tijdcontrole ongeveer een minuut later komen, plus de bezorgtijd.
- De status wordt **vóór** verzending opgeslagen om dubbele meldingen na een herstart te beperken. Een onderbroken of mislukte verzending telt als poging. Als het maximum nog niet is bereikt, volgt de volgende poging na het ingestelde interval.
- Verzending en wissen voor twee toestellen gebeuren in onafhankelijke parallelle takken. Een fout bij één toestel verhindert de poging voor het andere niet.
- Na het einde van de ophaaldag worden geen herhalingen meer verzonden.
- Handmatig resetten van de helper naar `{}` kan dezelfde dag een nieuwe cyclus starten wanneer tijd en sensordatum nog passen. Gebruik dit alleen bewust voor een nieuwe test.
- De eenmalige modus zonder bevestiging gebruikt alleen de ingestelde tijdtrigger; een gemist tijdstip wordt na een herstart niet ingehaald.

## Dank

Gebaseerd op de oorspronkelijke [Garbage reminder-blueprint uit de Home Assistant Community](https://community.home-assistant.io/t/garbage-reminder/284213). Dit project voegt opslag van antwoorden en automatische herhalingen toe.
