# HubSpot-Integration: Notes statt Custom Properties

> **Strategiewechsel (Phase B, Co-Work-Setup):** ursprünglich waren 5 Custom
> Properties auf dem Contact geplant. Stattdessen schreibt der
> `submit-handler.json`-Workflow alle Submission-Daten als **eine Note** an
> den Contact. Vorteil: kein Properties-Schema-Setup nötig, alle Vorhaben
> sind als Engagement-Verlauf am Contact sichtbar (auch historisch
> nachvollziehbar bei Wiederholungs-Workshops).

## Private-App-Setup

HubSpot → Settings → Integrations → Private Apps → **Create private app**.

**Erforderliche Scopes (reduziert gegenüber ursprünglichem Plan):**

| Scope | Wofür |
|-------|-------|
| `crm.objects.contacts.read`  | Existierenden Kontakt anhand E-Mail finden |
| `crm.objects.contacts.write` | Kontakt anlegen oder Standard-Felder updaten (Vorname, Nachname, Telefon) |
| `crm.objects.notes.write`    | Note anlegen + per Engagement an Contact assoziieren |

`crm.schemas.contacts.read` **entfällt** (kein Property-Setup mehr nötig).

Token einmalig nach Erstellung kopieren → in `.env` als
`HUBSPOT_PRIVATE_APP_TOKEN`.

## Workflow-Schritte (von `submit-handler.json` ausgeführt)

1. **Contact upserten** — `PATCH /crm/v3/objects/contacts/{email}?idProperty=email`
   mit Body `{ properties: { email, firstname, lastname, phone } }`. Liefert
   `contactId` zurück. Properties-Update beschränkt sich bewusst auf
   Standard-Felder; Workshop-/LIFO-Daten gehen ausschließlich in die Note.

2. **Note anlegen + assoziieren** — `POST /crm/v3/objects/notes` mit Body:

   ```json
   {
     "properties": {
       "hs_timestamp": "<ISO-Zeitstempel der Submission>",
       "hs_note_body": "<gerenderte Note, siehe Template unten>"
     },
     "associations": [
       {
         "to": { "id": "<contactId>" },
         "types": [{
           "associationCategory": "HUBSPOT_DEFINED",
           "associationTypeId": 202
         }]
       }
     ]
   }
   ```

   `associationTypeId: 202` = Note → Contact (HubSpot default association).

## Note-Body-Template

Plain-Text mit Zeilenumbrüchen (HubSpot rendert `\n` korrekt im
Activity-Feed). Variablen kommen aus dem Submit-Payload bzw. den per
Workshop-Lookup nachgeladenen SeaTable-Feldern.

```
{Workshop.Name} – {Workshop.Datum}
Trainer: {Trainer.Name}

Vorhaben (6 Wochen):
{Submission.Vorhaben6W}

Nächster Schritt (72h):
{Submission.NaechsterSchritt72h}

LIFO-Interesse: {Submission.LifoInteresse}
LIFO-Stärke: {Submission.LifoStaerke}
```

**Edge-Cases:**

- `LifoStaerke` ist immer leer (wird vom Webform nicht erfasst). Zeile
  trotzdem mit ausgeben (zeigt: nicht erfasst, nicht: Daten verloren).
- `LifoInteresse` aus dem Webform ist immer `ja` oder `nein` (binär). Wenn
  das Submit-Backend später um eine `vielleicht`-Quelle erweitert wird,
  funktioniert der String-Einsatz weiter.
- Workshop-Datum: deutsches Format `DD.MM.YYYY` (Workflow rendert es so).

## Was passiert NICHT mehr

- Keine Custom Properties auf Contact (`umsetzung_vorhaben_6w`,
  `umsetzung_naechster_schritt_72h`, `umsetzung_lifo_interesse`,
  `umsetzung_lifo_staerke`, `umsetzung_workshop_name`).
- Keine Property-Group `result_umsetzung` mehr.
- Kein Schema-Pre-Setup in HubSpot UI vor dem ersten Workflow-Run.
