# Master Data Import (Excel)

EEG admins can bulk-import members and metering points from an Excel workbook
("Stammdatenimport"). The import is started in the **web app → member pane →
upload button → "Stammdaten"**; the file is sent to the backend
(`POST /eeg/import/masterdata`, multipart fields `masterdatafile` and
`sheet` — the worksheet name the user enters), parsed there
(`database/excel.go`) and written into `base.participant` /
`base.meteringpoint`. A workbook may unzip to at most 128 MiB (backend 1.3.2);
larger files are rejected.

This page is the reference for the import template: which column ends up
where, which defaults apply, and how re-imports behave.

**Current template:**
[:material-download: 260716-vorlage-import-stammdaten.xlsx](../assets/260716-vorlage-import-stammdaten.xlsx)
— the cleaned successor of `250310-vorlage-import-stammdaten.xlsx`. It contains
only columns the importer actually reads; the import matches **header names,
not column positions**, so older templates keep working. The same file lives as
a pinned regression fixture in the backend repo
(`tests/260716-vorlage-import-stammdaten.xlsx`).

!!! info "Behavior by backend version"
    The date handling ("Mitglied seit", "registriert seit"), the
    "Zugeteilte Menge in Prozent" mapping, continue-on-error, the skipped-row
    notifications, the status/direction tolerance, the MitgliedsNr warnings and
    the enforced Gemeinschafts-ID ship with backend **1.1.0**. Older
    versions read a nonexistent "Dokument unterschrieben" column, set
    "Mitglied seit" to the import day, dropped date-formatted cells and always
    imported 100 % participation. Since **1.2.0** "Netzbetreiber" is optional
    and only compared; since **1.3.0** the optional "Einzugsart" column is read.

## Template structure

- Rows starting with `[### Leerzeile für Importer ###]` are decorative and skipped.
- The **header row** is the row whose first cell is `Netzbetreiber`, or that
  has a cell `Zählpunkt` (`isImportHeaderRow`). Column positions are derived
  from it (header names are matched case-insensitively), so column order may
  differ from the template.
- Every row below that has a value in "Zählpunkt", "Name 1" or "Name 2" is a
  **data row** — **one row per metering point**. A member with several
  metering points appears in several rows.
- Rows of one member are merged **by exact "Name 1" + "Name 2" match** (see
  [limitations](#limitations)).

## Field reference

The import matches columns by **header name** (letters below refer to the
current `260716` template).

| Column | Imported as | Notes |
|---|---|---|
| Netzbetreiber (A) | comparison only | optional. The metering point's `gridOperatorId` (and name) is derived from the Zählpunkt number, including the configured `grid-operator-alias` translation; a different value in this column is reported as warning `W_GRID_OPERATOR_IGNORED` |
| Gemeinschafts-ID (B) | validation only (not stored) | **required**: every row must carry the community id of the EEG you upload into (case-insensitive). A different id — the "wrong file / wrong community" mistake — or an **empty** cell rejects the row (reported once per distinct wrong id). Not checked if the EEG has no community id |
| PLZ / Ort / Straße / Hausnummer (C–F) | member resident + billing address **and** metering point address | one set of columns feeds all three |
| Zählpunkt (G) | metering point id | must be exactly 33 characters, otherwise the row is rejected (reported) |
| Energierichtung (H) | `CONSUMPTION` or `GENERATION` | empty defaults to `CONSUMPTION`; **any other value rejects the row (reported)** — a typo must not silently import a producer as consumer |
| EquipmentNr (I) | metering point `equipmentNumber` | |
| ObjektName (J) | metering point `equipmentName` | |
| Zugeteilte Menge in Prozent (K) | metering point participation factor (`partFact`) | plain number, `50%`, decimal comma and percent-formatted cells (raw `0.5` = 50 %) accepted; empty/invalid → 100. Legacy header `Teilnehmerfaktor`/`PartFact` still works |
| TitelVor / TitelNach (L, O) | member titles | |
| Name 1 / Name 2 (M, N) | member first name (or company name) / last name | if "Name 1" is empty or shorter than 2 characters, the importer tries to split "Name 2" as `<last> <first>`; failure is reported and the row skipped |
| BusinessRole (P) | `business` → `EEG_BUSINESS`, everything else → `EEG_PRIVATE` | |
| Mitglied seit (Q) | member `participantSince` | `d.m.yyyy` text **or** a real Excel date cell; empty → import day. Legacy header `Dokument unterschrieben` still works |
| IBAN / Kontoinhaber / Bankname (R–T) | member bank account | not validated |
| Einzugsart (not in template) | member direct debit type | optional, header `DebitType` works too: `CORE`, `B2B`, `NONE` or `Kein SEPA` (case, blanks, hyphens ignored). Empty/missing → no type, the member is **not debited** (one warning `W_DEBIT_TYPE_DEFAULT` counts new members with an IBAN but no type); unknown value → warning `W_DEBIT_TYPE_UNKNOWN`; a different type on a later row of the same member → `W_DEBIT_TYPE_CONFLICT` |
| MandatRef / MandatDat (not in template) | member SEPA mandate reference / date | optional; date as text or date cell |
| Email (U) | member e-mail | several addresses separated by `;` allowed; validated against the shared address rule; an invalid address is reported and the member imported **without** e-mail |
| TelefonNr (V) | member phone | not validated |
| SteuerNr / UmsatzsteuerNr (W, X) | member tax / VAT number | |
| MitgliedsNr (Y) | member number | taken from the member's **first** row; a number used by several members in the file or already assigned to a *different* existing member is reported as a **warning** (rows still import) |
| Zählpunktstatus (Z) | `ACTIVE` / `ACTIVATED` / `REGISTERED` → active member + active metering point; `NEW` → member and metering point process state `NEW`, metering point status `INIT` | case-insensitive, surrounding spaces tolerated; any other value rejects the row (reported) |
| registriert seit (AA) | metering point `registeredSince` and `activeSince` | text or date cell; empty → Jan 1 of the current year. For **online** (EDA-connected) EEGs `registeredSince` is always the import day (`activeSince` still comes from this column) |

**Not imported:** Columns from **older templates** without a data model
counterpart (Ortsgebiet, Stiege/Stock/Tür/Adresszusatz, Überschusseinspeisung,
Energiequelle, Verteilungsmodell, Meter Codes) are ignored; they were removed
from the `260716` template.

!!! tip "Where do I find my Gemeinschafts-ID?"
    It is the community id (`AT…`-33 characters) shown in the EEG master data
    ("Gemeinschafts-ID"). The template's sample row contains a placeholder id
    and is therefore always rejected on real imports — replace it with your
    own data.

## Adding metering points to an existing member { #re-import }

To attach an additional metering point to a member that already exists (whether
created manually or by an earlier import):

1. Take the import template and add **one row for the new metering point**.
2. Fill **"Name 1" and "Name 2" exactly as stored on the existing member**
   (character-for-character — the match is by name only).
3. Fill the metering point columns (Gemeinschafts-ID, Zählpunkt,
   Energierichtung, Zählpunktstatus, registriert seit, …) for the new ZP.
4. Upload the file for the same EEG.

The importer detects the existing member by name and **only appends the
metering points** of the row. All member-level columns of that row (IBAN,
e-mail, MitgliedsNr, Mitglied seit, address, …) are **ignored** — a re-import
never updates existing member master data. To change member data, edit the
member in the web app.

Rows for metering points the member already has (same active ZP id) fail with a
uniqueness error. Because each member is imported in one transaction, the
error rolls back **all** of that member's rows in the file (also its new metering
points); the member is reported in the import notification and the rest of the
file is still imported. Put only new metering points into a re-import file.

## Errors and notifications

The import never aborts halfway: each member is imported in its own
transaction. Problems are collected and delivered as an **admin notification**
("Excel Master Data Import") listing, per row: invalid e-mail addresses,
invalid `Zählpunktstatus`, unknown `Energierichtung`, missing/mismatching
`Gemeinschafts-ID`, metering point ids ≠ 33 characters, unextractable names,
and members whose database import failed. Duplicate `MitgliedsNr` values (within
the file, or already assigned to a different existing member), an ignored
`Netzbetreiber` value (`W_GRID_OPERATOR_IGNORED`) and direct debit type issues
(`W_DEBIT_TYPE_*`) appear as **warnings** — those rows are still imported.
The notification is only written when there is at least one message. Check the notification bell
after every import.

## Limitations { #limitations }

- **Members are matched by first + last name only.** Two different persons with
  the identical name in one EEG are merged into one member. The `MitgliedsNr`
  cannot serve as the key because legacy files number it *per row* (one member's
  rows may carry different numbers).
- Re-imports **never update** existing members' master data (by design — the
  import is an initial-load and ZP-append tool, not a sync).
- Existing files that left the Gemeinschafts-ID column empty must fill it in
  once — since the column is enforced, empty cells reject the row.
