# Quickscan HR — rapportgenerator (browser-app + n8n AI-proxy)

Een standalone tool voor consultants: vul de scan-gegevens in, upload de ingevulde quickscan-Excel, en download het opgemaakte Reijn-rapport. **Alles draait in de browser** — de Excel wordt lokaal verwerkt en de PDF lokaal opgebouwd. Alleen de rapporttekst wordt gegenereerd via een kleine n8n-webhook die Azure OpenAI aanroept (de enige stap die een geheim nodig heeft).

```
[Browser-app op GitHub Pages]
  Excel parsen + scores berekenen + PDF vullen (pdf-lib)   ← alles client-side
        │  POST { scan } + sleutel
        ▼
[n8n Cloud webhook]  Webhook → sleutelcheck → Basic LLM Chain (Azure) → Parse → Respond
        │  rapporttekst (JSON)
        ▼
  PDF wordt in de browser gevuld → download
```

## Bestanden
- `index.html` — de app (één bestand).
- `rapportsjabloon.pdf` — het lege Reijn-sjabloon (commit dit in dezelfde repo).
- `n8n-code-bouw-prompt.js` — Code-node die de prompt bouwt.
- `n8n-code-parse-ai.js` — Code-node die de JSON uit de AI-output haalt.

---

## Stap 1 — n8n-webhook bouwen (de AI-proxy)
Azure wordt aangeroepen via de **Basic LLM Chain**-node met een **Azure OpenAI Chat Model** eraan (geen HTTP Request nodig). Maak in n8n Cloud een workflow met deze nodes:

1. **Webhook** — Method `POST`, Path bijv. `quickscan-ai`, Respond = *Using 'Respond to Webhook' node*. Open **Options → Allowed Origins (CORS)** en zet je Pages-URL (bijv. `https://advanz80.github.io`) of `*` om te testen.
2. **IF** — sleutelcheck: `{{ $json.headers['x-app-key'] }}` *is gelijk aan* je geheim.
   - **False-tak →** een **Respond to Webhook** node, Response Code `401`.
3. **Code "Bouw prompt"** (true-tak) — plak `n8n-code-bouw-prompt.js`. Output: `json.prompt`.
4. **Basic LLM Chain** —
   - **Prompt** = *Define below*, en in het tekstveld: `{{ $json.prompt }}`
   - Hang er een **Azure OpenAI Chat Model** sub-node aan met je credential + deployment (gpt-4o).
   - *(Optioneel, voor strakkere JSON: zet "Require Specific Output Format" aan en koppel een Structured Output Parser. Niet nodig — de Parse-node hieronder vangt het op.)*
5. **Code "Parse AI"** — plak `n8n-code-parse-ai.js`. Haalt de JSON eruit, robuust voor de verschillende output-vormen van de chain (`{text}` of `{response:{text}}`) en code-fences. Output: `{themas, kansen}`.
6. **Respond to Webhook** — Response Body = `{{ $json }}`. Voeg in **Options → Response Headers** zo nodig `Access-Control-Allow-Origin: <jouw Pages-URL>` toe (als de CORS-optie van de Webhook dit niet al meestuurt).

Activeer de workflow en kopieer de **Productie-webhook-URL**.

## Stap 2 — App configureren
Open `index.html` en vul bovenin het `CONFIG`-blok:
```js
const CONFIG = {
  WEBHOOK_URL: 'https://<jouw>.app.n8n.cloud/webhook/quickscan-ai',
  STORAGE_KEY: 'reijn_quickscan_code',   // naam van de localStorage-sleutel
  TEMPLATE_URL:'./rapportsjabloon.pdf',
};
```
Er staat **geen toegangscode in het bestand**. De app toont bij openen een **inlogscherm**; de consultant voert daar de code in, die wordt in `localStorage` bewaard (dus per browser) en als `x-app-key`-header meegestuurd. n8n controleert 'm in de IF-node. Voor deze PoC deel je één gedeelde code uit; later kun je per consultant een eigen code geven zonder de app te wijzigen. Keurt n8n de code af (401), dan logt de app automatisch uit.

## Stap 3 — Publiceren op GitHub Pages
Commit `index.html` + `rapportsjabloon.pdf` naar je repo → **Settings → Pages** → branch `main` → `/root`. Na ~1 min staat de tool op `https://<gebruiker>.github.io/<repo>/`.

---

## Beveiliging (lees dit)
- **De code zit niet meer in de broncode** — de consultant voert 'm in bij de login en hij wordt in `localStorage` bewaard. Dat haalt het grootste bezwaar weg (een sleutel zichtbaar in de repo/paginabron). Let op: het blijft voor deze PoC **één gedeelde code**, dus lekt die, dan geldt dat voor iedereen. De volgende stap is per consultant een eigen code (zelfde mechanisme, een lijstje in n8n) en uiteindelijk echte auth vóór de app.
- **De webhook is een open kraan zonder slot.** De IF-sleutelcheck in n8n is de échte controle; kies een lange, willekeurige code. Zet daarnaast een **uitgavenlimiet op je Azure-deployment** en eventueel een daglimiet in n8n, zodat een gevonden webhook nooit een dure verrassing wordt.
- **CORS** in de Webhook-node moet je Pages-origin toestaan, anders blokkeert de browser de call.
- **GitHub Pages is publiek** (tenzij GitHub Enterprise met privé-Pages). Het lege sjabloon is dan publiek zichtbaar. Wil je de tool echt afschermen, host 'm dan op de Synology achter login.
- **AVG:** de ingevulde Excel verlaat de browser niet. Alleen de geparste scan-tekst gaat naar n8n→Azure (EU).

## Aandachtspunten in gebruik
- **Quote-velden kappen af.** Thema's 1/3/5/7 hebben een éénregelige oranje quote die niet wrapt; de prompt begrenst quotes daarom op ~12 woorden.
- **Cover-naam kort houden** — het veld op de voorkant is smal; de afdeling komt op pagina 3.
- **Sjabloon vast.** De PDF-veld-map en stercoördinaten zijn op dit sjabloon geijkt. Komt er een nieuw sjabloon, dan moeten die opnieuw gecontroleerd worden (`TV`, `CV`, `SY/SX` in `index.html`).

## Herbruikbaarheid
Deze tool is een sterke **Claude-skill-kandidaat** ("Reijn-rapport vullen"): de veld-map, stercoördinaten, scorelogica en prompt zijn stabiele domeinkennis. Werk je Skills/Preferences bij zodra je dit een paar keer hebt gedraaid.
