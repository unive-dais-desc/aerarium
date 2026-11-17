# Aerarium

DAIS and ISP-CNR environmental sample bank – atmosphere and cryosphere · Ca’ Foscari University of Venice and Institute of Polar Sciences (ISP-CNR)

*Banca dei campioni ambientali DAIS e ISP-CNR – atmosfera e criosfera*

Aerarium manages the archive of environmental samples – atmospheric particulate matter and gas phase (PUF), aqueous extracts, snow, sediments and laboratory degradation experiments – collected by DAIS and ISP-CNR and kept in the DAIS cold rooms, freezers and depots. It replaces the eLabJournal inventory previously used for the same purpose.

It has two parts:

- an **internal management application** for authorised staff, to find, register, move and retire samples;
- a **public website** on GitHub Pages, like Herbarium and Datasets Explorer, that shows the sampling campaigns on a map with summary data only.

The application is part of the tools developed by DESC – Data-centric Environmental Studies Centre, and shares its layout, typography and colours with Herbarium and Datasets Explorer.

## Architecture

```
Google Sheet (Campioni, Campagne, Siti, Contenitori, Utenti, Vocabolari, Registro)
      │ read / write                                   │ read only
      ▼                                                ▼
Apps Script project 1 ── Code.gs + App.html      Apps Script project 2 ── Public.gs
  web app, access: anyone within unive.it          web app, access: anyone
  = management application                         = JSON API with campaign summaries
                                                         │ fetch GET
                                                         ▼
                                              GitHub Pages ── index.html (public map)
                                                    └── "Staff login" button → management application
```

The two Apps Script projects are deliberately separate:

- The management application must know who is using it. Google tells the script the visitor's address only when the deployment is restricted to the domain.
- The public API must answer anyone, without sign-in, and contains only summaries. It cannot write to the sheet.

| File | Where it goes | Purpose |
|---|---|---|
| `Database Aerarium.xlsx` | Google Drive, converted to a Google Sheet | The database, migrated from eLabJournal |
| `Code.gs` | Apps Script project bound to the sheet | Management backend: access control, reads, writes, log |
| `App.html` | Same project, HTML file named `App` | Management interface |
| `Public.gs` | Separate, standalone Apps Script project | Public API: campaign summaries |
| `index.html` | GitHub repository (GitHub Pages) | Public map |

## Management application

**Access control**

| Role | Can do |
|---|---|
| `lettore` | Search, browse storage and map, read a sample's history, select samples and export them (CSV, Excel) |
| `editor` | All of the above, plus add, edit, move and retire samples, import samples from a file, and read the full log |
| `admin` | All of the above, plus create, edit and delete storage units, compartments and boxes |

- Users are listed in the **Utenti** sheet: `Email`, `Nome`, `Ruolo`, `Attivo` (write `no` to suspend a user).
- The account that deploys the script and the owner of the spreadsheet are always administrators. More administrators can be set in `ADMINS` in `Code.gs`.
- Anyone else sees a "Restricted access" page and receives no data: every backend function checks the role before reading or writing.
- Only accounts of the same Google Workspace domain as the deploying account (`unive.it`) can be recognised.

**Tabs**

- **Campioni** – search and filters (category, area, storage unit, campaign, sampling site, sample type, creator, contact point, status, sampling years, records to be checked), sample record with position, campaign, audit trail and history.
- **Stoccaggi** – tree of units, compartments and boxes; each box is drawn as its grid. Clicking a free cell registers a new sample there. With filters on, samples that do not match are dimmed.
- **Registro** – latest operations (editors and administrators).
- **Mappa** – the same map as the public website, built from all campaigns (including unpublished ones) and from the samples that pass the current filters. Clicking a site opens its campaigns; a campaign card shows internal notes and a button that lists its samples.

**Selection and export** (all roles)

A selection is built by adding samples from any view, in any combination:

- single samples, with the checkbox on each row or the button on the sample record;
- all the results of the current filters (campaign, site, unit, type, years… or any mix), with "Seleziona i N risultati"; change the filters and add again to combine groups;
- a whole unit, compartment or box, from **Stoccaggi**;
- a sampling site or a campaign, from **Mappa**.

When filters are active, the buttons in Stoccaggi and Mappa add only the samples that match them. A bar at the bottom shows the selection and lets you show only the selected samples, export them or clear the selection. The selection lasts until the page is reloaded.

Exports are available for the selection and for the current results:

- **CSV**: one row per sample with all its fields, campaign name and sites, and the full storage path.
- **Excel**: the same sheet, plus a **Campagne** sheet (campaigns involved, with contact points and internal notes) and a **Siti** sheet (sites with coordinates).

**Import from a file** (editors)

1. "Importa da file" → download the template, in Excel or CSV. It is generated at that moment, so menus and lists contain the current categories, campaigns and containers.
2. In the template, the header marks required fields with `*` (Riga and Colonna `(*)` only for grid boxes) and the second row says "obbligatorio" or "facoltativo". The third row is an example. At the bottom of the sheet are the instructions for every field and the lists of containers and campaigns. Rows starting with `#` and empty rows are ignored, so the example and the instructions can stay in the file. The Excel template also has drop-down menus and a **Valori** sheet with the allowed values.
3. Upload the completed file (.xlsx, or CSV in UTF-8 with comma or semicolon separator). The application checks every row with the same rules as the form and shows the errors row by row: occupied or repeated cell, position outside the grid, unknown container or campaign, invalid year, missing required field.
4. The import runs only when every row is valid, so a file is imported entirely or not at all. At most 500 samples per file. The imported samples are added to the selection.

Containers and campaigns can be written as an ID (`C004`, `CP10`), as the "ID – name" menu entry, or as the full name or storage path.

**Safeguards**

- A cell cannot hold two samples; a position outside the grid is refused.
- Retiring a sample (status `Smaltito`) frees its cell but keeps the record and its history.
- A box cannot be shrunk while samples would fall outside it; a container holding samples or other containers cannot be deleted.
- If two people edit the same sample, the second save is refused and asks to reload.
- Name, category and container are required; editors cannot change a sample's contact point; a sample can only be linked to a campaign that exists in the **Campagne** sheet.
- Every insertion, edit, move and status change is written to the **Registro** sheet.

## Public website

- Map with one base layer (OpenStreetMap). Each sampling site is a circle whose area is proportional to the number of samples; colours show the area (Arctic, Antarctica, Alpine, Urban).
- Clicking a site lists its campaigns; with a single campaign, its card opens directly.
- Campaign card: period, area, number of samples in storage and still usable, composition by category, sample types, size fractions, sampling sites, availability for external use, and the contact points of its samples with their email; a button writes to all of them with a pre-filled subject.
- Filters: free text, area, category, period, availability. Campaigns without coordinates are listed under the map's panel.
- "Staff login" button that opens the management application (set `ADMIN_URL`).
- Italian and English, light and dark theme, responsive layout. A campaign can be shared as a link (`#c=CP14`).

**What is published.** Only campaigns with `Pubblica` = `sì`. For each: name, public description, area, period, sites with coordinates, counts by category and type, size fractions, availability, contact points (name and email), link. Never published: individual samples and their names, storage units and positions, creators of the samples and who registered the records, internal notes, the "to be checked" column. Retired samples are not counted. Changes to the sheet appear on the website within 10 minutes.

## Data model

**Campioni** – one row per sample or batch.

| Column | Content |
|---|---|
| ID | Identifier assigned by the application (`AB-0001`…) |
| Nome, Categoria, Tipo campione | Sample name; Aerosol, Estratto acquoso, PUF, Sedimento, Degradazione artificiale, Neve; filter or support type |
| Area, Sito | Area of origin and site, as free text |
| Campagna | ID of the campaign in the **Campagne** sheet |
| Frazione, Anno da, Anno a, Quantità | Size fraction; sampling years; quantity as free text |
| Stato | Disponibile, Parzialmente utilizzato, Esaurito, Smaltito |
| Contenitore, Riga, Colonna | Container ID and position in its grid |
| Creatore | Who created the sample (internal use only, never published) |
| Referente | Who to contact for requests to use the sample; a name from the **Persone** sheet, published with its email. By default the same as the creator, for new samples too; only administrators can change it |
| Note, Da verificare | Notes, reason why the record needs checking |
| Creato da / il, Modificato da / il | Audit trail |

**Campagne** – one row per sampling campaign; the unit shown on the public website.

| Column | Content |
|---|---|
| ID, Nome | `CP01`…; campaign name |
| Descrizione pubblica | Text shown on the website |
| Area, Anno da, Anno a | If empty, derived from the samples |
| Siti | Site IDs separated by semicolons (`S09; S10`) |
| Pubblica | `sì` to show the campaign on the website |
| Disponibilità per uso esterno | Disponibile, Su richiesta, Non disponibile |
| Link | Optional page with more information (project, dataset DOI) |
| Note interne, Da verificare | Never published |

**Persone** – creators and contact points: `Nome` (as written in the Creatore and Referente columns), `Email`, `Note`, `Da verificare`. The email of a contact point appears on the public website. The campaign card lists the contact points of the campaign's samples.

**Siti** – sampling sites: `ID`, `Nome`, `Latitudine`, `Longitudine` (decimal degrees, negative = south / west), `Area`, `Paese`, `Note`, `Da verificare`.

**Contenitori** – the storage tree. `Padre` is the parent ID (empty for a unit); `Tipo` is `Unità`, `Scomparto` or `Box`; a box has `Righe` × `Colonne` cells labelled with numbers (`1`) or letters (`A`).

**Vocabolari** – suggested values for form fields. **Utenti** – authorised accounts. **Registro** – operations log.

Columns are recognised by name. Rows can be edited directly in the sheet, but manual edits are not logged; *Aerarium → Controllo di coerenza* reports duplicated cells, unknown containers, campaigns or sites, invalid coordinates and public campaigns without a contact email.

## Migration from eLabJournal

The database was built from the export `elabinventory_samples_2026_11_16.xlsx` (120 samples) and from the PDF printouts of the three storage units: 29 containers, 16 of them boxes with a grid.

- Each sample was placed in the cell shown in the PDFs; all 120 samples and all occupied cells match one to one.
- **Five positions must be checked on site**: samples with identical names in the same box (three named `2024` in Cold room -1 › Aerosols › Gruvebadet, two named `2014-2015 campioni 30^ spedizione` in Dosson › Antarctica › MZS). They are flagged in `Da verificare`.
- `Categoria`, `Area`, `Sito`, `Frazione` and the sampling years were derived from the compartment, the sample type and the sample name.
- **Campaigns and sites are a draft.** The 120 samples were grouped into 19 campaigns from their names and storage. The two `FR … 10 ppb` laboratory standards have no campaign. 14 sites have coordinates: Gruvebadet, Concordia and Mario Zucchelli are precise, the Italian sites are city centres. Availability is `Su richiesta` everywhere. The laboratory campaign is not public; the other 18 are.
- `Creatore` and `Referente` are the eLabJournal user; the **Persone** sheet lists the four users with an empty email, to be filled in.
- `Stato` is `Disponibile` for every sample, because eLabJournal does not record it.

The **Leggimi** sheet of the workbook repeats these notes.

## Setup

**Management application**

1. **Database.** Upload `Database Aerarium.xlsx` to Google Drive and choose File → Save as Google Sheets. In **Utenti**, add the authorised accounts.
2. **Backend.** In the sheet choose Extensions → Apps Script. Paste `Code.gs` over the default file. Add an HTML file named `App` and paste `App.html`.
3. **Deploy.** Deploy → New deployment → Web app, with *Execute as: Me* and *Who has access: Anyone within unive.it*. Copy the URL ending in `/exec`: it is the address of the management application.

**Public API**

4. Open [script.google.com](https://script.google.com) and create a **new, separate** project, for example "Aerarium – API pubblica". Paste `Public.gs` and set `SPREADSHEET_ID` to the ID of the sheet (the part of its address between `/d/` and `/edit`).
5. Deploy → New deployment → Web app, with *Execute as: Me* and *Who has access: Anyone*. Open `…/exec?action=getPublic` in a private window: it returns the public campaigns only.

**Public website**

6. In `index.html` set `API_URL` to the URL of step 5 and `ADMIN_URL` to the URL of step 3.
7. Upload `index.html` to a GitHub repository and enable GitHub Pages: Settings → Pages → Deploy from a branch → `main` / `(root)`.

After any change to a script, publish a new version of its existing deployment (Manage deployments → Edit → New version) so the URL stays the same.

## Configuration

| Setting | File | Purpose |
|---|---|---|
| `ADMINS` | `Code.gs` | Accounts that are always administrators |
| `ID_PREFIX` | `Code.gs` | Prefix of sample identifiers |
| `IMPORT_MAX` | `Code.gs` | Maximum number of samples per imported file (default 500) |
| `STATUS_GONE` | `Code.gs`, `Public.gs` | Statuses that free the cell and are not counted (default: `smaltito`) |
| `SHEETS`, `S_COLS`, `CP_COLS`, `SI_COLS`, `C_COLS` | `Code.gs` | Sheet names and column headers |
| `SPREADSHEET_ID` | `Public.gs` | ID of the sheet read by the public API |
| `FALLBACK_CONTACT` | `Public.gs` | Contact shown for campaigns whose samples have no contact point |
| `STATUS_USABLE` | `Public.gs` | Statuses counted as "usable" (default: Disponibile, Parzialmente utilizzato) |
| `CACHE_TTL` | `Public.gs` | Seconds before a change to the sheet reaches the website (default 600) |
| `API_URL`, `ADMIN_URL` | `index.html` | Public API and management application addresses |

## Third-party services and privacy

The pages load Google Fonts (Barlow, Spectral, IBM Plex Mono), Leaflet 1.9.4 from cdnjs, OpenStreetMap tiles and the DESC logo from `unive.it`; the ISP-CNR logo is embedded in the pages. They set no cookies and use no analytics; the browser stores only the chosen interface language (`localStorage`). The public website publishes the name and email of the contact points (Persone sheet): fill in the emails only with each person's agreement. Creators are never published. The management application records the Google address of whoever inserts or edits a sample.

## Licence

© 2026 DAIS – Ca’ Foscari University of Venice

This project is licensed under the [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International licence (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/). The DESC, Ca’ Foscari and ISP-CNR logos, OpenStreetMap data and third-party libraries are not covered by this licence.
