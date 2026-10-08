# BW Woo Membership

WordPress-Plugin für **Vereinsmitgliedschaften mit Jahresbeitrag** auf Basis von WooCommerce und
**WooCommerce Subscriptions**. Erster Einsatz: **APPA – Austrian Positive Psychology Association**
(geplant 500–1.000+ Mitglieder).

Das Plugin ergänzt WooCommerce Subscriptions um die Vereinsregeln, die kein fertiges Plugin abbildet:
Kalenderjahr-Mitgliedschaft, Beitragsstaffel nach Beitrittsmonat, Kündigung nur zum Jahresende mit Frist,
Freigabe bestimmter Mitgliedsarten nach Nachweis sowie Mitgliedsarten mit unterschiedlichen Leistungen.

> **Status:** Spezifikation. Diese README ist die verbindliche Sammlung aller Anforderungen und
> Entscheidungen aus der Konzeptphase (Oktober 2026). Code folgt in den Arbeitspaketen unten.

---

## Inhalt

1. [Entscheidungen](#1-entscheidungen)
2. [Voraussetzungen](#2-voraussetzungen)
3. [Fachliche Regeln](#3-fachliche-regeln)
4. [Mitgliedsarten, Aufnahme und Leistungen](#4-mitgliedsarten-aufnahme-und-leistungen)
5. [Statusmodell](#5-statusmodell)
6. [Abläufe](#6-abläufe)
7. [Zahlung](#7-zahlung)
8. [Funktionsumfang des Plugins](#8-funktionsumfang-des-plugins)
9. [Technisches Konzept](#9-technisches-konzept)
10. [E-Mails](#10-e-mails)
11. [Betrieb bei 500–1.000+ Mitgliedern](#11-betrieb-bei-5001000-mitgliedern)
12. [Datenschutz und Recht](#12-datenschutz-und-recht)
13. [Testplan](#13-testplan)
14. [Arbeitspakete und Aufwand](#14-arbeitspakete-und-aufwand)
15. [Offene Punkte](#15-offene-punkte)
16. [Umgebung und verwandte Repos](#16-umgebung-und-verwandte-repos)

---

## 1. Entscheidungen

| Thema | Entscheidung | Begründung |
|---|---|---|
| Basis | **WooCommerce + WooCommerce Subscriptions (WCS)** | Standard für wiederkehrende Zahlungen in WordPress; synchronisierte Verlängerung auf ein festes Datum (1. 1.) eingebaut; offizielle Stripe- und PayPal-Plugins sind darauf abgestimmt; keine Provision. Lizenz **279 USD/Jahr** (wird von blickwert gekauft). |
| Vereinslogik | **Eigenes Plugin `bw-woo-membership`** | Staffelpreis, Kündigungsfrist, Freigabe und Leistungen je Mitgliedsart deckt kein fertiges Plugin ab. |
| Zahlungsarten | **nur PayPal, Apple Pay, Google Pay, Kreditkarte** | Nur diese erlauben automatische wiederkehrende Abbuchung. Keine Banküberweisung, kein SEPA-Lastschrift-Start. |
| Beitragsstaffel | **Variante B** (siehe 3.2) | Entscheidung APPA/blickwert. |
| Dezember-Beitritt | Dezember gratis, voller Beitrag fürs Folgejahr (Abbuchung am 1. 1.) | Entscheidung. |
| Aufnahme | ordentlich: sofort nach Zahlung; assoziiert/studentisch: nach Freigabe (Nachweis) | Entscheidung. |
| Nicht verwendet | WooCommerce Memberships, Paid Memberships Pro, Mollie | Leistungen/Rechte macht das eigene Plugin; PMPro wäre ein zweites Kassensystem neben WooCommerce. |
| Geprüft und verworfen | **Milo Subscriptions** (Stand 08.10.2026) | Seit 13.05.2026 am Markt, 200 Installationen; keine Synchronisierung auf ein festes Datum (nur Add-on „Scheduled Start Dates“); eigenes Zahlungs-Plugin mit **1 % Provision** (bei ~1.000 × 100 € ≈ 1.000 €/Jahr) oder ohne Provision keine automatische PayPal-Abbuchung. Späterer Wechsel bleibt möglich (gleicher Datentyp `shop_subscription`, Migrator vorhanden). |
| Austauschbarkeit | Abo-Engine hinter einer Adapter-Schicht | WCS-spezifische Aufrufe gekapselt (`includes/engine/`), damit ein Wechsel (z. B. zu Milo) nur den Adapter betrifft. |

## 2. Voraussetzungen

| Komponente | Version / Hinweis | Kosten |
|---|---|---|
| WordPress | ≥ 6.4 | – |
| PHP | ≥ 7.4 (Ziel 8.2) | – |
| WooCommerce | ≥ 8.0, HPOS-kompatibel | kostenlos |
| WooCommerce Subscriptions | aktuelle Version, Option „Synchronise renewals“ aktiv | 279 USD/Jahr |
| WooCommerce Stripe Gateway | Kreditkarte, Apple Pay, Google Pay (Domain-Verifizierung für Apple Pay) | kostenlos + Transaktionsgebühren |
| WooCommerce PayPal Payments | PayPal mit Vaulting (gespeicherte Zahlungsfreigabe) | kostenlos + Transaktionsgebühren |
| PDF Invoices & Packing Slips for WooCommerce | Beitragsbestätigungen als PDF | kostenlos |
| Elementor + Elementor Pro, Theme Hello Elementor/Hello Biz | Mitgliederbereich, Dynamic Tags, Anzeigebedingungen | vorhanden |
| Server-Cron | echter Cronjob statt WP-Cron (`DISABLE_WP_CRON`) | – |

Zeitzone der Website: **Europe/Vienna** (Dev-Server steht derzeit auf Europe/Berlin – umstellen; alle
Stichtage werden in der WordPress-Zeitzone berechnet).

## 3. Fachliche Regeln

### 3.1 Laufzeit

- Eine Mitgliedschaft gilt immer für ein **Kalenderjahr: 1. Jänner bis 31. Dezember**.
- Verlängerung **automatisch** um ein Kalenderjahr; Abbuchung des **vollen Jahresbeitrags am 1. Jänner**.

### 3.2 Beitragsstaffel (Variante B) – Erstbeitrag nach Beitrittsmonat

| Beitritt im | Erstbeitrag | Gültig bis | Nächste Abbuchung |
|---|---|---|---|
| Jänner – Juni | 100 % | 31. 12. laufendes Jahr | 1. 1. Folgejahr, 100 % |
| Juli – September | 50 % | 31. 12. laufendes Jahr | 1. 1. Folgejahr, 100 % |
| Oktober – November | ⅓ (33,33 %) | 31. 12. laufendes Jahr | 1. 1. Folgejahr, 100 % |
| Dezember | 0 € (Dezember gratis) | 31. 12. **Folgejahr** | 1. 1. Folgejahr, 100 % (deckt das Folgejahr) |

- Maßgeblich ist das **Datum der Aktivierung**: bei ordentlicher Mitgliedschaft das Bestelldatum, bei
  Freigabe-Mitgliedsarten das Datum der Bestellung nach der Freigabe.
- Stufen (Startmonat, Prozentsatz) sind **im Admin einstellbar** und können **je Mitgliedsart** abweichen.
- Rundung: kaufmännisch auf 2 Nachkommastellen (z. B. ⅓ von 100 € = 33,33 €).
- Der Warenkorb/Checkout zeigt verständlich: „Beitrag 2026 (anteilig ab Oktober): 33,33 € – ab 1. 1. 2027
  jährlich 100 €“.

### 3.3 Kündigung

- Kündigung **nur zum Jahresende**, Kündigungsfrist **ein Monat** → spätester Termin **30. November**
  (23:59 Uhr, Zeitzone Wien).
- Kündigung bis 30. 11.: Mitgliedschaft bleibt **bis 31. 12. aktiv**, danach keine Abbuchung mehr
  (WCS-Status „Kündigung vorgemerkt“ / `pending-cancel`).
- Kündigung ab 1. 12.: wirkt erst zum **31. 12. des Folgejahres**. Die Abbuchung am 1. 1. erfolgt noch;
  das Plugin merkt die Kündigung vor und setzt sie nach der Verlängerung um. Das Konto zeigt das Datum klar an.
- Kündigungswege: **Button im Mitgliederkonto** oder wie bisher **per E-Mail an info@appa.at**
  (Admin trägt sie mit einem Klick ein). Beide erzeugen eine **Bestätigung per E-Mail** an Mitglied und Verein.
- Admin kann jederzeit abweichend kündigen (z. B. sofort, bei Ausschluss oder Todesfall) – mit Notiz.
- Kündigungsfrist und Stichtag sind einstellbar.

## 4. Mitgliedsarten, Aufnahme und Leistungen

### 4.1 Mitgliedsarten (Start)

| Mitgliedsart | Zielgruppe | Nachweis | Aktiv ab |
|---|---|---|---|
| Ordentlich | Psycholog:innen mit Hochschulabschluss und österreichischem Bezug | keiner beim Beitritt | **sofort nach Zahlung** |
| Assoziiert | Berufstätige mit Bezug zur Positiven Psychologie (Pädagogik, Sozialarbeit, Coaching, Beratung) | Upload (Berufs-/Ausbildungsnachweis) | **nach Freigabe** |
| Studentisch | Psychologiestudierende in Österreich | Upload Inskriptionsbestätigung | **nach Freigabe** |

- Jede Mitgliedsart = ein WooCommerce-Abo-Produkt (jährlich, synchronisiert auf 1. 1.) + Plugin-Einstellungen
  (Freigabe nötig ja/nein, Nachweis-Feld, Staffel, Leistungen).
- **Weitere Mitgliedsarten** (z. B. Fördermitglied, Institution) lassen sich später **ohne Programmierung**
  anlegen.
- Wechsel der Mitgliedsart (z. B. studentisch → ordentlich nach Abschluss): Admin-Funktion, wirkt zum
  nächsten 1. 1.; anteilige Differenz im laufenden Jahr wird nicht verrechnet (einstellbar).

### 4.2 Aufnahme mit Freigabe (assoziiert, studentisch)

1. Mitglied stellt online einen **Antrag** (Konto anlegen, Mitgliedsart wählen, Nachweis hochladen).
2. Bis zur Freigabe sieht es ein **Basiskonto** mit dem Hinweis: „Ihr Nachweis wird geprüft. Nach der
   Überprüfung wird Ihre Mitgliedschaft freigeschaltet.“
3. APPA sieht offene Anträge in der **Antragsliste** und gibt frei oder lehnt ab (mit optionaler Begründung).
4. **Freigabe** → E-Mail mit persönlichem Link „Mitgliedschaft jetzt abschließen“ → Checkout → Zahlung →
   aktiv. **Ablehnung** → E-Mail; Angebot einer anderen Mitgliedsart möglich.
5. Annahme (zu bestätigen, siehe [Offene Punkte](#15-offene-punkte)): **Zahlung erst nach Freigabe** –
   keine Rückerstattungen nötig.
6. Freigabe-Mitgliedsarten sind ohne Freigabe **nicht kaufbar** (Produkt für den Benutzer gesperrt).
7. Optional: Nachweis jährlich erneut anfordern (z. B. Inskription bei studentisch) – einstellbar, Standard aus.

### 4.3 Leistungen je Mitgliedsart (Rechte)

Jede Mitgliedsart vergibt eine Liste von **Rechten**. Alles andere (Seiten, Widgets, Preise) fragt nur Rechte ab,
nie Mitgliedsarten direkt.

| Recht (Schlüssel) | Bedeutung | Beispiel |
|---|---|---|
| `member` | aktives Mitglied (jede Art) | Mitgliederbereich |
| `downloads` | Dokumente & Downloads | Tagungsunterlagen, Statuten, Jahresbericht |
| `recordings` | Aufzeichnungen | Vortragsvideos |
| `directory_listing` | Eintrag im öffentlichen Mitgliederverzeichnis | nur mit Zustimmung (Opt-in) |
| `event_discount` | vergünstigte Veranstaltungen | Mitgliederpreis auf Event-Produkte |
| `voting` | Stimmrecht (für ordentliche Mitglieder) | Generalversammlung |

- API: `bw_member_can( string $right, ?int $user_id = null ): bool`
- Shortcodes: `[bw_member_only right="downloads"]…[/bw_member_only]`, `[bw_member_field field="status"]`
- Elementor Pro: **Dynamic Tags** und **Anzeigebedingungen** („nur für Recht X“).
- Inhalte schützen: Seiten/Beiträge/Downloads per Metabox auf ein Recht beschränken.

## 5. Statusmodell

Eigene Mitgliedschafts-Status, zugeordnet zu WCS-Status:

| Plugin-Status | Bedeutung | WCS-Abo | Rechte aktiv |
|---|---|---|---|
| `applied` (beantragt) | Antrag mit Nachweis eingereicht | – | nein (Basiskonto) |
| `approved` (freigegeben) | freigegeben, Zahlung ausstehend | – / `pending` | nein |
| `rejected` (abgelehnt) | Antrag abgelehnt | – | nein |
| `active` (aktiv) | bezahlt, läuft | `active` | **ja** |
| `on_hold` (Wartestellung) | Abbuchung fehlgeschlagen, Wiederholungen laufen | `on-hold` | ja, bis Ende der Kulanzfrist (einstellbar, Standard 30 Tage) |
| `cancel_scheduled` (Kündigung vorgemerkt) | gekündigt, läuft bis 31. 12. | `pending-cancel` | ja |
| `cancel_next_year` (Kündigung zum Folgejahr) | nach 30. 11. gekündigt | `active` | ja; nach Verlängerung → `cancel_scheduled` |
| `ended` (beendet) | ausgelaufen, gekündigt oder unbezahlt | `cancelled` / `expired` | nein |

Historie: jede Statusänderung mit Datum, Auslöser (Mitglied/Admin/System) und Notiz.

## 6. Abläufe

```
Beitritt online
├─ ordentlich ──────────────► Checkout ─► Zahlung (Staffelpreis) ─► aktiv bis 31.12.
└─ assoziiert / studentisch ─► Antrag + Nachweis ─► Basiskonto
                                 └─ Freigabe durch APPA ─► Link ─► Checkout ─► Zahlung ─► aktiv

jedes Jahr:
aktiv ─► bis 30.11. gekündigt? ── ja ──► endet 31.12.
                │ nein
                ▼
        1.1.: automatische Abbuchung voller Beitrag
                ├─ erfolgreich ─► läuft ein weiteres Jahr
                └─ scheitert ──► Wartestellung, Wiederholungen + Erinnerung
                                  └─ weiter erfolglos ─► Liste „offene Beiträge“ ─► nach Kulanzfrist beendet
```

## 7. Zahlung

- Erlaubte Zahlungsarten: **Kreditkarte, Apple Pay, Google Pay** (Stripe) und **PayPal** (PayPal Payments,
  Vaulting). Andere Zahlungsarten werden für Mitgliedschaftsprodukte ausgeblendet.
- Zahlungsart wird beim Beitritt einmal bestätigt (inkl. 3-D Secure); Folgezahlungen laufen ohne Zutun.
- Mitglied kann die **Zahlungsart selbst ändern** (Mitgliederkonto, WCS „Zahlungsmethode ändern“).
- Fehlgeschlagene Zahlung: WCS-Wiederholungssystem (Retry) aktiv, Zeitplan prüfen/anpassen;
  Erinnerungs-E-Mails mit „Jetzt bezahlen“-Link.
- Beiträge sind voraussichtlich **nicht umsatzsteuerbar** (echte Mitgliedsbeiträge) → Produkte steuerfrei;
  **Bestätigung durch Steuerberatung offen**.
- Zu jeder Zahlung **Beitragsbestätigung als PDF** (E-Mail + Konto), fortlaufend nummeriert.
- Stripe- und PayPal-Konten lauten auf den **Verein**; Auszahlungen gesammelt aufs Vereinskonto.

## 8. Funktionsumfang des Plugins

### Mitglied (Frontend)

- [ ] Beitritt: Mitgliedsart wählen, Staffelpreis-Anzeige, Checkout mit erlaubten Zahlungsarten
- [ ] Antrag mit Nachweis-Upload für Freigabe-Mitgliedsarten, Basiskonto mit Hinweis
- [ ] Mitgliederkonto: Status, Mitgliedsart, Mitglied seit, gültig bis, nächste Abbuchung (Datum + Betrag)
- [ ] Kündigen-Button mit Fristlogik und klarer Datumsanzeige
- [ ] Zahlungsart ändern, Rechnungen/Beitragsbestätigungen herunterladen
- [ ] **Mitgliedsbestätigung als PDF** (aktuelles Jahr)
- [ ] Profil/Sichtbarkeit im Mitgliederverzeichnis (Opt-in)
- [ ] Mitgliederbereich der APPA-Website (Elementor) mit echten Daten: Dynamic Tags für Vorname,
      Status, Mitgliedsart, Mitglied seit, nächste Verlängerung; Bedingungen nach Recht

### APPA (Admin)

- [ ] Einstellungen: Mitgliedsarten ↔ Produkte, Freigabe ja/nein, Staffel je Art, Kündigungsfrist/Stichtag,
      Kulanzfrist, Rechte je Art, E-Mail-Texte
- [ ] **Antragsliste** mit Nachweis-Ansicht, Freigeben/Ablehnen per Klick
- [ ] **Mitgliederliste** mit Filter (Art, Status, Jahr), Suche, Statushistorie, manuelle Aktionen
      (kündigen, Art wechseln, Notiz)
- [ ] **Offene Beiträge**: alle in Wartestellung/unbezahlt, mit Erinnerung erneut senden
- [ ] **Jahresexport CSV** (Mitglied, Art, Zeitraum, Betrag, Zahlungsart, Bestell-/Abo-Nr., Datum) für die
      Buchhaltung
- [ ] Kennzahlen: aktive Mitglieder je Art, Kündigungen zum Jahresende, offene Beiträge
- [ ] **Mitgliederrabatt** auf Veranstaltungs-Produkte (Recht `event_discount`, Rabatt % oder Fixpreis je Produkt)
- [ ] Import bestehender Mitglieder (CSV) mit Einladung zum Hinterlegen der Zahlungsart – **Umfang offen**

## 9. Technisches Konzept

### 9.1 Struktur (geplant)

```
bw-woo-membership.php          Bootstrap, Konstanten, Abhängigkeitsprüfung (WooCommerce, WCS)
includes/
  class-plugin.php             Init, Hooks
  class-install.php            Tabellen, Migrationen (dbDelta + maybe_migrate)
  class-membership-types.php   Mitgliedsarten (Konfiguration, Rechte)
  class-memberships.php        Datensätze, Status, Historie
  class-pricing.php            Staffelpreis (reine Funktion + WC-Filter)
  class-cancellation.php       Kündigungsfrist, Vormerkung, Umsetzung nach Verlängerung
  class-applications.php       Antrag, Nachweis-Upload (geschützt), Freigabe/Ablehnung
  class-rights.php             bw_member_can(), Inhaltsschutz, Shortcodes
  class-account.php            Mein Konto (WooCommerce My Account Endpoints)
  class-emails.php             E-Mails (WC_Email-Klassen)
  class-certificate.php        Mitgliedsbestätigung PDF
  class-export.php             CSV-Export
  class-event-discount.php     Mitgliederrabatt
  elementor/                   Dynamic Tags, Anzeigebedingungen
  engine/
    interface-engine.php       Adapter-Schnittstelle Abo-Engine
    class-engine-wcs.php       Umsetzung für WooCommerce Subscriptions
  admin/                       Einstellungen, Listen (WP_List_Table)
templates/                     überschreibbar im Theme (bw-woo-membership/)
languages/                     Textdomain bw-woo-membership, de_DE
tests/                         Unit-Tests ohne WordPress (wie bw-credits-booking)
tools/                         run-tests.php, make-pot.php …
```

### 9.2 Datenmodell

Tabelle `{prefix}bwm_memberships` (eine Zeile pro Mitglied und Mitgliedschaft):

| Spalte | Typ | Beschreibung |
|---|---|---|
| `id` | BIGINT | PK |
| `user_id` | BIGINT | WP-Benutzer |
| `type` | VARCHAR(32) | Mitgliedsart (Schlüssel) |
| `status` | VARCHAR(24) | siehe Statusmodell |
| `subscription_id` | BIGINT NULL | WCS-Abo |
| `member_since` | DATE | erstes Aktivdatum |
| `valid_until` | DATE NULL | Ende der aktuellen Laufzeit |
| `cancel_requested_at` | DATETIME NULL | Eingang der Kündigung |
| `cancel_effective` | DATE NULL | Wirksamkeit (31. 12. lfd. Jahr oder Folgejahr) |
| `created_at` / `updated_at` | DATETIME | |

Tabelle `{prefix}bwm_applications`: `id, user_id, type, status (pending/approved/rejected), proof_file,
note, decided_by, decided_at, created_at`.

Tabelle `{prefix}bwm_log`: `id, membership_id, event, actor (member/admin/system), data JSON, created_at`.

Nachweis-Dateien: in einem **nicht öffentlichen Verzeichnis** (`wp-content/uploads/bwm-private/` mit
`.htaccess deny` bzw. Auslieferung nur über PHP mit Rechteprüfung); Löschung nach Entscheidung einstellbar.

### 9.3 Staffelpreis mit WooCommerce Subscriptions

- Produkte: Typ „Einfaches Abo“, Intervall **jährlich**, **synchronisiert auf 1. Jänner**,
  WCS-Einstellung „Prorate first renewal“ = **nie**.
- Der **Erstbetrag** wird vom Plugin gesetzt (Filter auf den Preis der Erstbestellung), der **wiederkehrende
  Betrag** bleibt der volle Jahresbeitrag. Dezember: Erstbetrag 0 €, erste Abbuchung am 1. 1.
- Die Staffel ist eine **reine, testbare Funktion**: `bwm_initial_price(float $annual, DateTimeImmutable $date, array $tiers): float`.
- **Zu verifizieren** (früh im Arbeitspaket): genaue WCS-Hooks für den synchronisierten Erstbetrag
  (z. B. `woocommerce_subscriptions_product_price` / Sign-up-Fee-Weg / Warenkorb-Filter) und dass der
  1.-1.-Termin bei Beitritt am 31. 12. und 1. 1. korrekt liegt.

### 9.4 Kündigung mit WCS

- Kunden-Aktion „Kündigen“ im Konto über WCS-Filter steuern: bis 30. 11. → `pending-cancel` (Ende = nächster
  Zahlungstermin 1. 1.), ab 1. 12. → Plugin-Status `cancel_next_year`, WCS-Abo bleibt `active`.
- Nach erfolgreicher Verlängerung am 1. 1. setzt das Plugin vorgemerkte Kündigungen auf `pending-cancel`.
- WCS-Aktionen, die die Regeln umgehen würden (sofort kündigen, pausieren, Wechsel durch Kunden), werden für
  Mitgliedschaftsprodukte ausgeblendet.

### 9.5 Wiederverwendung aus `bw-credits-booking`

| Baustein | Fundstelle in bw-credits-booking |
|---|---|
| Freischaltung bei „Bestellung abgeschlossen“ inkl. Schutz gegen Doppelausführung, Rücknahme bei Storno | `handle_order_completed()` / `handle_order_reversed()` in `bw-credits-booking.php` |
| Eigene Tabellen + Migration | `dbDelta`, `maybe_migrate()` in `bw-credits-booking.php` |
| Produkt-Zusatzfelder im Admin | `includes/admin.php` (`woocommerce_product_options_general_product_data`) |
| E-Mail-Modul mit Cron und Sprache | `includes/emails.php`, `includes/email-language.php` |
| Konto-Dashboard | `render_account_dashboard()` |
| Settings-Seite, Template-Overrides | `includes/settings.php`, `includes/templates.php` |
| Auto-Updater über GitHub-Releases | `includes/updater.php` |
| Übersetzung, Tests | `tools/` (make-pot, make-mo, run-tests), `tests/` |

Die optionale PMPro-Anbindung in `bw-credits-booking/includes/membership.php` wird durch einen Filter dieses
Plugins ersetzt (`bwm_has_active_membership`), damit auch das Buchungs-Plugin Mitgliedschaften erkennt.

### 9.6 Code-Konventionen

- Präfix `bwm_` / Klassen `BWM_`, Textdomain `bw-woo-membership`, alle Texte übersetzbar (de_DE mitliefern).
- WooCommerce HPOS-kompatibel (`FeaturesUtil::declare_compatibility`).
- Alle Datums-Berechnungen mit `wp_timezone()` und `DateTimeImmutable`.
- Keine Ausgabe ohne Escaping, alle Admin-Aktionen mit Nonce + Capability (`manage_woocommerce`).

## 10. E-Mails

Als WooCommerce-E-Mail-Klassen (Texte im Admin anpassbar, APPA-Design):

| E-Mail | an | Auslöser |
|---|---|---|
| Willkommen / Mitgliedschaft aktiv | Mitglied | erste erfolgreiche Zahlung |
| Antrag eingegangen | Mitglied + APPA | Antrag mit Nachweis |
| Antrag freigegeben (mit Link zum Abschluss) | Mitglied | Freigabe |
| Antrag abgelehnt | Mitglied | Ablehnung |
| Erinnerung Freigabe-Link | Mitglied | X Tage nach Freigabe ohne Abschluss |
| Hinweis vor Abbuchung | Mitglied | z. B. 14 Tage vor 1. 1. (Betrag, Zahlungsart, Kündigungshinweis) |
| Beitrag abgebucht + PDF-Bestätigung | Mitglied | Verlängerung erfolgreich |
| Zahlung fehlgeschlagen / Jetzt bezahlen | Mitglied | Abbuchung gescheitert (WCS-Retry) |
| Kündigung bestätigt (mit Wirksamkeitsdatum) | Mitglied + APPA | Kündigung |
| Mitgliedschaft beendet | Mitglied | Ende |

## 11. Betrieb bei 500–1.000+ Mitgliedern

- Alle Verlängerungen am 1. 1. laufen über den **Action Scheduler** von WCS in Paketen → **echter Server-Cron**
  nötig.
- Laufende Aufgaben der APPA: Anträge freigeben, Liste „offene Beiträge“ prüfen, Jahresexport an die Buchhaltung.
- Mit einigen Prozent fehlgeschlagener Abbuchungen pro Jahr ist zu rechnen (abgelaufene Karten, 3-D Secure).
- Health-Checks im Admin: Mitglieder ohne gültige Zahlungsart vor dem 1. 1. auflisten und anschreiben.

## 12. Datenschutz und Recht

- Mitgliederverzeichnis nur mit ausdrücklicher Zustimmung (Opt-in, jederzeit widerrufbar).
- Datenschutzerklärung um Stripe und PayPal ergänzen.
- Nachweis-Dateien geschützt speichern, nach Entscheidung löschen (einstellbar).
- DSGVO: Export/Löschung personenbezogener Daten über die WordPress-Datenschutz-Werkzeuge anbinden.
- Kündigungsweg „Klick im Konto + E-Mail-Bestätigung“ und Frist gegen Statuten prüfen (APPA).

## 13. Testplan

Automatisierte Tests (ohne WordPress, wie `bw-credits-booking/tests`):

- Staffelpreis für jeden Monat, Grenzen: 30. 6./1. 7., 30. 9./1. 10., 30. 11./1. 12., 31. 12./1. 1.,
  Schaltjahr, Rundung ⅓.
- Kündigung: 30. 11. 23:59 vs. 1. 12. 00:00 (Wien), Wirksamkeitsdatum, Vormerkung nach Verlängerung.
- Status-Übergänge inkl. Wartestellung und Kulanzfrist.

Testläufe im Stripe- und PayPal-Testmodus auf dem Dev-Server:

- Beitritt je Mitgliedsart und Staffel, Apple Pay/Google Pay (Stripe-Testkarten), PayPal-Sandbox.
- Verlängerung am 1. 1. (simuliert über „Zahlung jetzt verarbeiten“), fehlgeschlagene Zahlung + Retry,
  Zahlungsart ändern, Kündigung vor/nach Stichtag, Freigabe/Ablehnung.

## 14. Arbeitspakete und Aufwand

Geschätzt mit voller KI-Unterstützung, gesamt **38 h** (Stundensatz 96 € netto → 3.648 € netto).

| # | Arbeitspaket | Stunden |
|---|---|---|
| 1 | Detailkonzept (Statusmodell, Stichtage, E-Mail-Texte, Abstimmung) | 3 |
| 2 | Shopeinrichtung (WooCommerce, WCS, Stripe inkl. Apple/Google Pay, PayPal, PDF, Produkte, Kasse/Konto im APPA-Design, E-Mail-Vorlagen, Rechtstexte, Cronjob, Testbestellungen) | 8 |
| 3 | Plugin-Grundgerüst (Daten, Mitgliedsarten, Einstellungen, Updates) | 2 |
| 4 | Beitragsstaffel | 2 |
| 5 | Kündigung | 2 |
| 6 | Antrag und Freigabe | 4 |
| 7 | Leistungen je Mitgliedsart | 2 |
| 8 | Mitgliederbereich (Elementor-Anbindung, PDF-Bestätigung) | 3 |
| 9 | Verwaltung (offene Beiträge, Jahresexport) | 2 |
| 10 | Mitgliederrabatt Veranstaltungen | 1 |
| 11 | Tests (automatisiert + Stripe/PayPal-Testmodus) | 4 |
| 12 | Übernahme bestehender Mitglieder (**zu klären**) | 2 |
| 13 | Abnahme und Einschulung | 3 |
|   | **Summe** | **38** |

Laufende Kosten: WCS-Lizenz 279 USD/Jahr + Transaktionsgebühren Stripe/PayPal.
Intern ca. 10 h Puffer einplanen; Regeländerungen während der Umsetzung kosten zusätzliche Stunden.

## 15. Offene Punkte

Entscheidungen der APPA:

- [ ] Jahresbeitrag je Mitgliedsart (ordentlich, assoziiert, studentisch)
- [ ] Assoziiert/studentisch: Zahlung **erst nach Freigabe** (Vorschlag) oder schon beim Antrag
- [ ] Umsatzsteuer: Bestätigung der Steuerberatung (Mitgliedsbeiträge nicht umsatzsteuerbar)
- [ ] Statuten: Aufnahme, Kündigungsfrist und Kündigungsweg passen?
- [ ] Datenschutz: Opt-in Mitgliederverzeichnis, Datenschutzerklärung (Stripe, PayPal)
- [ ] Übernahme bestehender Mitglieder: nötig? (im Briefing aus den offenen Punkten entfernt, im Aufwand noch enthalten)

Zugänge (auf den Verein lautend):

- [ ] Stripe-Konto der APPA
- [ ] PayPal-Geschäftskonto mit aktivierter gespeicherter Zahlungsfreigabe (Vaulting)

Technisch (blickwert):

- [ ] WCS-Lizenz kaufen und auf dem Dev-Server installieren
- [ ] Zeitzone Dev-Server auf Europe/Vienna
- [ ] Genaue WCS-Hooks für Erstbetrag und Kündigungssteuerung verifizieren (9.3, 9.4)
- [ ] WCS-Retry-Zeitplan prüfen (Fremdquelle: ab Werk aus, 5 Versuche in 7 Tagen)

## 16. Umgebung und verwandte Repos

| | |
|---|---|
| Dev-Server | https://dev.blickwert.at/wp/2603-appa/ (WordPress, Elementor Pro, Hello Biz, BW WP Bridge) |
| Zugriff für Claude Code | Plugin **BW WP Bridge** + Umgebungsvariablen `WP_URL`, `WP_USER`, `WP_APP_PASSWORD`; Domain in den Allowed domains der Cloud-Umgebung |
| `blickwert/appa-data` | Mockup und Elementor-Templates der APPA-Website (inkl. Mitgliederbereich-Seiten) |
| `blickwert/bw-wp-bridge` | REST-Bridge + Client `tools/wp_bridge.py` |
| `blickwert/bw-credits-booking` | Buchungs-Plugin, Vorlage für Architektur und Bausteine |
| Kundenbriefing | „APPA – Mitgliedschaft online: Konzept & Empfehlung“ (Claude-Dokument, Stand 08.10.2026) |
