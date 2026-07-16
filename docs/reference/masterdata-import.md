# Master Data Import (Excel)

EEG admins can bulk-import members and metering points from an Excel workbook
("Stammdatenimport"). The import is started in the **web app → member pane →
upload button → "Stammdaten"**; the file is sent to the backend
(`POST /eeg/import/masterdata`), parsed there
(`database/excel.go`) and written into `base.participant` /
`base.meteringpoint`.

This page is the reference for the import template
(`250310-vorlage-import-stammdaten.xlsx`): which column ends up where, which
defaults apply, and how re-imports behave.

!!! info "Behavior as of backend PR #34"
    The date handling ("Mitglied seit", "registriert seit"), the
    "Zugeteilte Menge in Prozent" mapping, continue-on-error and the skipped-row
    notifications described here ship with the first backend release after
    v1.0.7. Older versions read a nonexistent "Dokument unterschrieben" column,
    set "Mitglied seit" to the import day, dropped date-formatted cells and
    always imported 100 % participation.

## Template structure

- Rows starting with `[### Leerzeile für Importer ###]` are decorative and skipped.
- The **header row** is the row whose first cell is `Netzbetreiber`. Column
  positions are derived from it, so column order may differ from the template.
- Every row below whose column A matches a grid-operator id (`AT` + digits,
  e.g. `AT003000`) is a **data row** — **one row per metering point**. A member
  with several metering points appears in several rows.
- Rows of one member are merged **by exact "Name 1" + "Name 2" match** (see
  [limitations](#limitations)).

## Field reference

| Column | Imported as | Notes |
|---|---|---|
| Netzbetreiber (A) | metering point `gridOperatorId` (+ name lookup) | row trigger; must match `AT`+digits, surrounding spaces are tolerated |
| PLZ / Ort / Straße / Hausnummer (D–G) | member resident + billing address **and** metering point address | one set of columns feeds all three |
| Zählpunkt (L) | metering point id | must be exactly 33 characters, otherwise the row is rejected (reported) |
| Energierichtung (M) | `CONSUMPTION` or `GENERATION` | anything else silently falls back to `CONSUMPTION` |
| EquipmentNr (N) | metering point `equipmentNumber` | |
| ObjektName (O) | metering point `equipmentName` | |
| Zugeteilte Menge in Prozent (S) | metering point participation factor (`partFact`) | plain number, `50%`, decimal comma and percent-formatted cells (raw `0.5` = 50 %) accepted; empty/invalid → 100. Legacy header `Teilnehmerfaktor`/`PartFact` still works |
| TitelVor / TitelNach (T, W) | member titles | |
| Name 1 / Name 2 (U, V) | member first name (or company name) / last name | if "Name 1" is empty, the importer tries to split "Name 2" as `<last> <first>`; failure is reported and the row skipped |
| BusinessRole (X) | `business` → `EEG_BUSINESS`, everything else → `EEG_PRIVATE` | |
| Mitglied seit (Y) | member `participantSince` | `d.m.yyyy` text **or** a real Excel date cell; empty → import day. Legacy header `Dokument unterschrieben` still works |
| IBAN / Kontoinhaber / Bankname (Z–AB) | member bank account | not validated |
| Email (AC) | member e-mail | validated against the shared address rule; an invalid address is reported and the member imported **without** e-mail |
| TelefonNr (AD) | member phone | not validated |
| SteuerNr / UmsatzsteuerNr (AE, AF) | member tax / VAT number | |
| MitgliedsNr (AG) | member number | taken from the member's **first** row; no uniqueness check |
| Zählpunktstatus (AH) | `ACTIVE` / `ACTIVATED` / `REGISTERED` → active member + active metering point; `NEW` → new, not yet activated | exact match, case-sensitive; any other value rejects the row (reported) |
| registriert seit (AI) | metering point `registeredSince` and `activeSince` | text or date cell; empty → Jan 1 of the current year. For **online** (EDA-connected) EEGs `registeredSince` is always the import day |

**Not imported** (template columns without a data model counterpart):
Gemeinschafts-ID (B — also *not* validated against the target EEG!), Ortsgebiet
(C), Stiege/Stock/Tür/Adresszusatz (H–K), Überschusseinspeisung (P),
Energiequelle (Q), Verteilungsmodell (R), Meter Codes (AJ).

## Adding metering points to an existing member { #re-import }

To attach an additional metering point to a member that already exists (whether
created manually or by an earlier import):

1. Take the import template and add **one row for the new metering point**.
2. Fill **"Name 1" and "Name 2" exactly as stored on the existing member**
   (character-for-character — the match is by name only).
3. Fill the metering point columns (Zählpunkt, Energierichtung,
   Zählpunktstatus, registriert seit, …) for the new ZP.
4. Upload the file for the same EEG.

The importer detects the existing member by name and **only appends the
metering points** of the row. All member-level columns of that row (IBAN,
e-mail, MitgliedsNr, Mitglied seit, address, …) are **ignored** — a re-import
never updates existing member master data. To change member data, edit the
member in the web app.

Rows for metering points the member already has (same active ZP id) fail with a
uniqueness error; that row is reported in the import notification and the rest
of the file is still imported.

## Errors and notifications

The import never aborts halfway: each member is imported in its own
transaction. Problems are collected and delivered as an **admin notification**
("Excel Master Data Import") listing, per row: invalid e-mail addresses,
invalid `Zählpunktstatus`, metering point ids ≠ 33 characters, rows without a
valid `Netzbetreiber`, unextractable names, and members whose database import
failed. Check the notification bell after every import.

## Limitations { #limitations }

- **Members are matched by first + last name only.** Two different persons with
  the identical name in one EEG are merged into one member. The `MitgliedsNr`
  cannot serve as the key because legacy files number it *per row* (one member's
  rows may carry different numbers).
- Re-imports **never update** existing members' master data (by design — the
  import is an initial-load and ZP-append tool, not a sync).
- The Gemeinschafts-ID column is not checked against the EEG you upload into —
  double-check you are logged into the right community before uploading.
