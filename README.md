# Umsetzungstool — result. learning &amp; transfer

Webform + Backend für die Nachverfolgung von Workshop-Vorhaben.
Teilnehmer:innen scannen am Ende eines Workshops einen QR-Code, halten
in 3 Schritten ihr 6-Wochen-Vorhaben fest, und bekommen 4 zeitversetzte
Erinnerungs-Mails von ihrem Trainer.

**Live-Domain:** [https://umsetzung.result-lt.de](https://umsetzung.result-lt.de)

---

## Architektur in einem Bild

```
                ┌──────────────────────────────────┐
                │  https://umsetzung.result-lt.de  │
                │  (Cloudflare Pages, Vanilla JS)  │
                └────────────────┬─────────────────┘
                                 │ HTTPS
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
     ┌────────────────┐  ┌────────────────┐  ┌─────────────────┐
     │ get-active-    │  │ get-workshop-  │  │ submit-form     │
     │ workshops      │  │ by-id          │  │ (POST)          │
     │ (n8n webhook)  │  │ (n8n webhook)  │  │ (n8n webhook)   │
     └────────┬───────┘  └────────┬───────┘  └────────┬────────┘
              │                   │                   │
              └────────┬──────────┘                   │
                       ▼                              ▼
              ┌────────────────────────┐    ┌─────────────────┐
              │  SeaTable (Workshops,  │◄───┤  + HubSpot      │
              │  Trainer, Submissions, │    │    (Note + Upsert)
              │  ScheduledEmails)      │    └─────────────────┘
              └────────────┬───────────┘
                           │ Cron alle 15 min
                           ▼
              ┌────────────────────────┐    ┌─────────────────┐
              │  email-sender (n8n)    │───►│  MS Graph       │
              │  pollt pending Mails   │    │  umsetzung@…    │
              └────────────────────────┘    └─────────────────┘
```

**Tech-Stack** (final, nicht ändern ohne Diskussion):

| Schicht        | Tool                     | Rolle                                       |
|----------------|--------------------------|---------------------------------------------|
| Webform        | Vanilla HTML/CSS/JS      | Single-File `index.html`, kein Build-Step   |
| Hosting        | Cloudflare Pages         | Free-Tier, Auto-Deploy via Git, CNAME       |
| Datenhaltung   | SeaTable Cloud           | Base „Umsetzungs-Tracking", 4 Tabellen      |
| Orchestrierung | n8n (selbst gehostet)    | 4 Workflows                                 |
| CRM            | HubSpot Free             | Notes via REST API (kein Custom-Properties) |
| E-Mail         | MS Graph                 | App-Permission Mail.Send aus umsetzung@…    |

---

## Verantwortlichkeiten

Wer macht was — 4 Rollen, klare Grenzen.

### 🟦 DU (Daniel)

Alles was **Tokens, Zugänge, finale Entscheidungen** braucht.

- HubSpot Private App anlegen → Token an Co-Work
- Azure App-Registration „Umsetzungstool Mailer" anlegen → Tenant-ID + Client-ID + Client-Secret an Co-Work
- DNS-Eintrag `umsetzung.result-lt.de CNAME <project>.pages.dev` setzen lassen
- Inhaltliche Freigaben (Tonalität der Mails, Trainer-Foto-URLs)
- End-to-end-Test nach Live-Schaltung

### 🎨 Claude Design — ✅ erledigt

UI-Prototyp gebaut (`index.html`), result.-Brand übernommen, 3-Step-Wizard
+ 2 Confirmation-Varianten + Tweaks-Panel.

### 💻 Claude Code — ✅ erledigt (diese Phase)

Code, Workflows, Templates. Was hier liegt ist deployment-ready bis auf
die Platzhalter (`{{...}}`), die Co-Work setzt.

### 🌐 Claude Co-Work

Alles was **im Browser geklickt wird**.

- SeaTable: Base anlegen, 4 Tabellen + Spalten nach
  [setup/seatable-schema.json](setup/seatable-schema.json), Trainer-Stammdaten
  + Test-Workshop befüllen → ✅ erledigt
- n8n: 4 Workflows aus `workflows/*.json` importieren, Credentials
  konfigurieren, Webhook-URLs kopieren
- HubSpot: Private App mit reduzierten Scopes anlegen (siehe
  [setup/hubspot-note-template.md](setup/hubspot-note-template.md))
- Azure: bei DU rückfragen für Mail-App-Registration, dann n8n-
  OAuth2-Credential konfigurieren
- Cloudflare: Pages-Projekt anlegen + Custom-Domain konfigurieren
- Find-and-Replace `{{N8N_…_URL}}` in `index.html` durch echte URLs

---

## Erst-Setup — vollständige Reihenfolge

Beim ersten Mal hochziehen, danach nur noch Workshops anlegen.

### 1. 🌐 SeaTable-Base prüfen

Sollte schon stehen (Co-Work hat das bei Phase A erledigt). Verifikation:

- Base **Umsetzungs-Tracking** existiert, UUID `36f34c52-dcdd-49a6-b414-976465d26ee4`
- Tabellen: Trainer, Workshops, Submissions, ScheduledEmails
- 4 Trainer befüllt (mit `Email`, `FotoUrl`, `SignatureHtml`, `Aktiv = true`)
- Mindestens 1 Workshop mit Status `active`
- API-Token erstellt → in `.env` als `SEATABLE_API_TOKEN`

Schema-Referenz: [setup/seatable-schema.json](setup/seatable-schema.json).

### 2. 🟦 HubSpot Private App anlegen

HubSpot → Settings → Integrations → Private Apps → **Create private app**.

Erforderliche Scopes (genau diese drei, nicht mehr):

- `crm.objects.contacts.read`
- `crm.objects.contacts.write`
- `crm.objects.notes.write`

Token einmalig kopieren → in `.env` als `HUBSPOT_PRIVATE_APP_TOKEN`.

Details: [setup/hubspot-note-template.md](setup/hubspot-note-template.md).

### 3. 🟦 Azure App-Registration „Umsetzungstool Mailer"

Azure Portal → Azure Active Directory → App registrations → **New
registration**.

- **Name:** Umsetzungstool Mailer
- **Supported account types:** Single tenant (result.)
- **Redirect URI:** keine

Nach Anlage:

- API permissions → Add → Microsoft Graph → Application permissions →
  `Mail.Send` → **Grant admin consent**
- Optional aber empfohlen: Application Access Policy via PowerShell auf
  genau die Mailbox `umsetzung@result.de` einschränken (sonst kann die
  App theoretisch aus jeder Mailbox senden):

  ```powershell
  Connect-ExchangeOnline
  New-ApplicationAccessPolicy `
    -AppId <CLIENT_ID> `
    -PolicyScopeGroupId umsetzung@result.de `
    -AccessRight RestrictAccess `
    -Description "Umsetzungstool darf nur aus umsetzung@result.de senden"
  ```

- Certificates &amp; secrets → New client secret → Wert kopieren
  (wird nur einmal angezeigt)

→ `MS_GRAPH_TENANT_ID`, `MS_GRAPH_CLIENT_ID`, `MS_GRAPH_CLIENT_SECRET`
in `.env` eintragen.

### 4. 🌐 n8n-Workflows importieren

In n8n: Workflows → Import from File → für jeden der 4 Workflows.

| Datei | Webhook-Pfad |
|---|---|
| [workflows/get-active-workshops.json](workflows/get-active-workshops.json) | `/webhook/get-active-workshops` |
| [workflows/get-workshop-by-id.json](workflows/get-workshop-by-id.json) | `/webhook/get-workshop-by-id` |
| [workflows/submit-handler.json](workflows/submit-handler.json) | `/webhook/submit-form` |
| [workflows/email-sender.json](workflows/email-sender.json) | (Cron, kein Webhook) |

**Credentials konfigurieren** (3 Stück):

- **`{{SEATABLE_CRED_ID}}`** → Credential-Typ „HTTP Header Auth"
  - Name: `Authorization`
  - Value: `Bearer <SEATABLE_API_TOKEN>`
- **`{{HUBSPOT_CRED_ID}}`** → „HTTP Header Auth"
  - Name: `Authorization`
  - Value: `Bearer <HUBSPOT_PRIVATE_APP_TOKEN>`
- **`{{MS_GRAPH_CRED_ID}}`** → „OAuth2 API" (Generic)
  - Grant Type: **Client Credentials**
  - Access Token URL: `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token`
  - Client ID: `<MS_GRAPH_CLIENT_ID>`
  - Client Secret: `<MS_GRAPH_CLIENT_SECRET>`
  - Scope: `https://graph.microsoft.com/.default`
  - Authentication: „Send as Body"

In jedem Workflow die `{{…_CRED_ID}}`-Platzhalter durch die echten
Credential-IDs ersetzen (n8n-UI → Credential im jeweiligen Node neu
auswählen).

**Webhook-URLs kopieren** (nach dem ersten Speichern jedes Webhook-
Workflows zeigt n8n die finale URL):

- `N8N_GET_WORKSHOPS_URL` → in `.env` + in `index.html`
- `N8N_GET_WORKSHOP_BY_ID_URL` → in `.env` + in `index.html`
- `N8N_SUBMIT_URL` → in `.env` + in `index.html`

Workflows auf **active** schalten — nur die 3 Webhook-Workflows. Den
`email-sender` erst NACH einem erfolgreichen Test-Submit aktivieren
(damit nicht sofort Test-Mails rausgehen).

### 5. 💻 `index.html` patchen

Die 3 Webhook-Platzhalter ersetzen:

```bash
sed -i '' \
  -e 's|{{N8N_GET_WORKSHOPS_URL}}|https://n8n.result.de/webhook/get-active-workshops|' \
  -e 's|{{N8N_GET_WORKSHOP_BY_ID_URL}}|https://n8n.result.de/webhook/get-workshop-by-id|' \
  -e 's|{{N8N_SUBMIT_URL}}|https://n8n.result.de/webhook/submit-form|' \
  index.html
```

(Pfade entsprechend eurer n8n-Domain anpassen.)

### 6. 🌐 Cloudflare Pages

1. **GitHub-Repo verbinden**: Cloudflare Dashboard → Workers &amp; Pages
   → Create → Pages → Connect to Git → Repo wählen.
2. **Build settings**: Framework = `None`, Build command leer, Build
   output directory = `/`.
3. **Deploy** → Cloudflare gibt eine `<project>.pages.dev`-URL.
4. **Custom Domain**: Pages → Custom domains → Add → `umsetzung.result-lt.de`.
   Cloudflare zeigt das CNAME-Target an.

### 7. 🟦 DNS

Bei eurem DNS-Hoster (oder Cloudflare DNS, falls die Zone dort liegt):

```
umsetzung   CNAME   <project>.pages.dev
```

SSL-Zertifikat wird automatisch von Cloudflare ausgestellt
(~5–10 min nach DNS-Propagierung).

### 8. End-to-end-Test

```bash
# Webform erreichbar?
curl -I https://umsetzung.result-lt.de
# → HTTP/2 200

# Test-Submit (Workshop-ID aus SeaTable nehmen):
curl -X POST https://n8n.result.de/webhook/submit-form \
  -H 'Content-Type: application/json' \
  -d '{
    "workshop_id": "<SEATABLE_WORKSHOP_ROW_ID>",
    "participant_name": "Test User",
    "participant_email": "deine-test@adresse.de",
    "participant_phone": null,
    "vorhaben_6w": "Test-Vorhaben.",
    "naechster_schritt_72h": "Test-Schritt.",
    "lifo_interest": "nein",
    "lifo_staerke": null,
    "consent": true
  }'
# → 200 + {ok:true, trainer_name, participant_name}
```

Verifikation:

- SeaTable Submissions: 1 neue Row mit deinen Test-Daten
- SeaTable ScheduledEmails: 3 (oder 4 mit LIFO) Rows mit Status `pending`
- HubSpot Contacts: Test-Kontakt vorhanden
- HubSpot Activity-Feed dieses Kontakts: 1 Note mit Workshop-Daten

Erst jetzt den **email-sender-Workflow aktivieren**. Innerhalb von 15 min
sollte die `thanks_24h`-Mail in der Test-Inbox sein (SendAt ist 24h in
der Zukunft — zum sofortigen Test SendAt manuell in SeaTable auf
Vergangenheit setzen).

---

## Neuen Workshop einrichten (1-Minuten-Prozess)

Für jeden Workshop, der das Tool nutzen soll:

1. **SeaTable** öffnen → Base „Umsetzungs-Tracking" → Tabelle
   **Workshops**.
2. Neue Row anlegen:
   - **Name**: z.B. „KI in Führung — Nov 2026"
   - **Datum**: Workshop-Datum
   - **Trainer**: aus Dropdown wählen (Hauptverantwortliche:r zuerst)
   - **LifoVerwendet**: ✓ wenn LIFO® im Workshop eingesetzt wurde
   - **Status**: `active`
3. Nach dem Speichern werden die Formelspalten **automatisch** befüllt:
   - **FormUrl**: fertige Form-URL inkl. `?workshop_id=…`
   - **QrCodeImage**: generierter QR-Code zur FormUrl
4. **QR-Code in den Workshop-Slide einbauen**:
   Rechtsklick auf das QrCodeImage → Bild speichern → in die
   Abschluss-Folie einfügen.
5. **Alternativ** (digital): die FormUrl per Mail oder Chat verschicken.

Nach dem Workshop: in SeaTable Status auf `archived` ändern, sobald die
6-Wochen-Reflexionsmail rausging. Verhindert versehentliche
Spät-Submissions auf alte Codes.

---

## E-Mail-Templates editieren

**Source-of-Truth**: [emails/](emails/) — diese Dateien werden im Browser
gepflegt (mit beliebigem HTML-Editor) und sind so wie sie liegen
preview-bar.

Nach jedem Edit der `emails/*.html`-Dateien müssen die Templates **in
den email-sender-Workflow re-embedded** werden:

```bash
# Im Repo-Root ausführen:
python3 <<'PY'
import json
TYPES = ['thanks_24h', 'followup_78h', 'final_6w', 'lifo_followup']
templates = {t: open(f'emails/{t}.html').read() for t in TYPES}

with open('workflows/email-sender.json') as f:
    wf = json.load(f)

esc = lambda s: s.replace('\\', '\\\\').replace('`', '\\`').replace('${', '\\${')
entries = ',\n  '.join(f'{n}: `{esc(c)}`' for n, c in templates.items())

for node in wf['nodes']:
    if node['id'] == 'node-render-012':
        code = node['parameters']['jsCode']
        idx = code.find('\n\n// === DATEN SAMMELN ===')
        head = code[:code.find('// === EMAIL TEMPLATES ===')]
        new_block = (
            "// === EMAIL TEMPLATES ===\n"
            "// Source-of-Truth: emails/*.html — siehe README → Templates editieren.\n"
            "\n"
            "const TEMPLATES = {\n  " + entries + "\n};"
        )
        node['parameters']['jsCode'] = head + new_block + code[idx:]
        break

with open('workflows/email-sender.json', 'w', encoding='utf-8') as f:
    json.dump(wf, f, indent=2, ensure_ascii=False)
print('Re-embed OK.')
PY
```

Dann den Workflow in n8n manuell neu importieren (oder über die n8n-API
patchen).

**Variablen-Convention** (gleich in allen Templates):

| Platzhalter | Quelle | HTML-escaped? |
|---|---|---|
| `{{FIRSTNAME}}` | TeilnehmerName, erstes Wort | ja |
| `{{FULLNAME}}` | TeilnehmerName | ja |
| `{{WORKSHOP_NAME}}` | Workshops.Name | ja |
| `{{WORKSHOP_DATUM}}` | Workshops.Datum, dd.mm.yyyy | ja |
| `{{VORHABEN_6W}}` | Submission.Vorhaben6W | ja (Newlines→`<br>`) |
| `{{NAECHSTER_SCHRITT_72H}}` | Submission.NaechsterSchritt72h | ja |
| `{{TRAINER_NAME}}` | Trainer.Name | ja |
| `{{TRAINER_EMAIL}}` | Trainer.Email | ja |
| `{{TRAINER_PHOTO_URL}}` | Trainer.FotoUrl | ja |
| `{{TRAINER_SIGNATURE}}` | Trainer.SignatureHtml | **nein** (rohes HTML) |
| `{{QR_CODE_IMAGE}}` | Workshops.QrCodeImage | ja |
| `{{FORM_URL}}` | Workshops.FormUrl | ja |

---

## Trouble-Shooting

| Symptom | Wahrscheinliche Ursache | Lösung |
|---|---|---|
| Webform: „Workshops nicht verfügbar" | n8n nicht erreichbar / CORS | n8n-Webhook im Browser-Devtools → Network testen. CORS-Header in `Respond to Webhook`-Node prüfen (`Access-Control-Allow-Origin`). |
| Workshop nicht im Dropdown | Status ≠ `active` | In SeaTable Workshop-Row auf Status `active` setzen. |
| `?workshop_id=…` zeigt „nicht (mehr) verfügbar" | ID falsch oder Workshop archiviert | SeaTable: Row-ID prüfen + Status. |
| Submit hängt im Loading-State | n8n-Workflow `submit-form` inaktiv ODER Credentials fehlerhaft | n8n → Workflow → Executions → letzten Run anschauen. |
| Submission angelegt, aber HubspotContactId leer | HubSpot-Token abgelaufen / Scopes falsch | Token in HubSpot regenerieren. |
| Mails kommen nicht an | email-sender inaktiv ODER MS-Graph-Cred falsch | n8n → email-sender → Executions. SeaTable ScheduledEmails.Status auf `failed` → ErrorMessage lesen. |
| MS-Graph-Fehler „MailboxNotEnabledForRESTAPI" | App-Permission ohne Admin-Consent | In Azure: API permissions → Grant admin consent. |
| Mails landen im Spam | SPF/DKIM für `result.de`-Domain fehlt für umsetzung@-Versand | DNS-Setup für die Mailbox prüfen. |

---

## Datei-Index

```
.
├── README.md                          ← du bist hier
├── CLAUDE.md                          (Agent-Instructions, framework-doc)
├── .gitignore
├── index.html                         ← das Webform (single file, 50 KB)
├── assets/
│   ├── logo.png
│   └── logo-small.png
├── fonts/
│   └── Roboto-Regular.ttf
├── emails/                            ← Mail-Templates (source-of-truth)
│   ├── thanks_24h.html
│   ├── followup_78h.html
│   ├── final_6w.html
│   └── lifo_followup.html
├── workflows/                         ← n8n-Workflows (importierbar)
│   ├── get-active-workshops.json
│   ├── get-workshop-by-id.json
│   ├── submit-handler.json
│   └── email-sender.json
└── setup/                             ← Setup-Doku + Schemata
    ├── seatable-schema.json
    ├── hubspot-note-template.md
    └── .env.example
```

---

## Phase-A vs. Phase-B Historie

- **Phase A** (vor diesem Repo): Setup-Definitionen für SeaTable, HubSpot
  und MS Graph. Co-Work hat parallel die Base + Trainer + Test-Workshop
  in SeaTable angelegt, einige Schema-Abweichungen entstanden
  (Trainer.FotoUrl statt Foto-Image-Upload, ScheduledEmails.Subject als
  Key-Spalte, alle Links is_multiple=true). HubSpot-Strategie auf Notes
  statt Custom Properties geändert.
- **Phase B** (dieses Repo): Backend-Anbindung in `index.html` ergänzt,
  4 n8n-Workflows als JSON, 4 E-Mail-Templates, Hosting-Setup.

Commit-History (`git log --oneline`) zeigt jeden Schritt einzeln —
Phase B wurde in 9 atomic Commits geliefert.
