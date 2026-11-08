# Phase 1: Kern – Konzept

Stand: 2026-11-08 · Status: **Entwurf zur Freigabe** · Grundlage: [Gesamtkonzept](overall-concept.md) und [Entscheidungen 0001](../decisions/0001-overall-concept-decisions.md)

Dieses Dokument legt fest, was in Phase 1 gebaut wird und wie. Was im Gesamtkonzept schon entschieden ist, wird hier nur noch verfeinert und nicht neu begründet; Verweise wie „GK 3.4.3“ zeigen auf den Abschnitt im Gesamtkonzept. Quellen stehen in Abschnitt 17 ([K1] usw.). Offene Entscheidungen für Christian stehen in Abschnitt 16 (K-E1 usw.).

---

## 1. Ziel und Umfang

**Ziel:** Ein lauffähiger, getesteter Kern, auf dem in Phase 2 das erste Plugin (Einkaufslisten) aufsetzen kann, ohne dass der Kern dafür geändert werden muss.

**Im Umfang:**

1. Monorepo, Build, Lint, Tests, CI.
2. Lokale Test- und Entwicklungsumgebung mit HTTPS im Heimnetz.
3. Konten, Einladungen, Anmeldung (Passkey; Passwort plus TOTP), Wiederherstellungscodes, Sitzungen, Geräte.
4. Gruppen, Rollen, Rechte.
5. Plugin-Host mit Manifest, Signaturprüfung, Datenbereichen, Lebenszyklus; Signierwerkzeug.
6. Sync-Engine mit Konfliktlogik, Live-Aktualisierung.
7. PWA-Grundgerüst: Shell, Service Worker, verschlüsselter Offline-Speicher, Entsperren, Warteschlange.
8. i18n (de, en).
9. Benachrichtigungen (Web Push, Mail über Relay).
10. Jobs, Audit-Log, Alarme, Admin-Bereich.
11. Ein **Test-Plugin** („demo-notes“, nur in Entwicklung und Tests), das alle Plugin-Schnittstellen benutzt und beweist, dass der Kern ohne echtes Plugin vollständig testbar ist.

**Nicht im Umfang:** fachliche Plugins, CalDAV (Phase 3), Backup (Phase 4), Produktivbetrieb und Härtung des echten Servers (Phase 5). Der Kern bekommt aber schon die Schnittstellen, die diese Phasen brauchen (z. B. Backup-Hooks, Alarmkanäle).

**Grundsatz „schlank zuerst“:** Jede Funktion wird zuerst in der kleinsten sinnvollen Form gebaut und getestet. Was hier als „später“ markiert ist, kommt erst nach einem Test im Alltag.

---

## 2. Technik und Abhängigkeiten

### 2.1 Grundgerüst

| Bereich | Wahl | Bemerkung |
|---|---|---|
| Sprache | TypeScript, `strict` | Code, Bezeichner, Kommentare, Logs auf Englisch (GK 3.6). |
| Laufzeit | Node.js, aktuelle LTS | Genaue Version wird beim Projektstart festgelegt und in `.nvmrc` und `engines` festgeschrieben. |
| Paketverwaltung | npm mit Workspaces | Kein zusätzliches Werkzeug nötig. Lockfile wird eingecheckt, Installation in CI mit `npm ci`. |
| Web-Framework | Fastify | Validierung über JSON Schema mit Ajv, Logging mit Pino, beides eingebaut [K1][K2]. |
| Datenbank | PostgreSQL, Treiber `pg` | Eigene, kleine Migrationslogik (SQL-Dateien mit Nummer), keine ORM-Bibliothek. |
| UI | **Vorschlag: Svelte** (ohne SvelteKit), gebaut mit Vite (Entscheidung K-E1) | Kompiliert zu schlankem JavaScript; Alternativen Lit oder Preact. |
| Tests | Vitest (Unit, Integration), Playwright (Ende-zu-Ende) | Playwright steuert echte Browser-Engines, auch WebKit, die Engine von Safari. Das ersetzt keinen Test mit Safari auf einem echten iPhone; der kommt in den Akzeptanzkriterien dazu (14). |

### 2.2 Etablierte Bibliotheken für Sicherheit und Protokolle

Freigegeben sind etablierte Bibliotheken für Krypto, Datenbank, Web-Framework und CalDAV (GK 1). Für den Kern sind das:

| Zweck | Bibliothek | Warum keine Eigenentwicklung |
|---|---|---|
| Passkeys (WebAuthn) | `@simplewebauthn/server` und `@simplewebauthn/browser` [K3] | Prüfung von Attestation und Signaturen ist fehleranfällig und sicherheitskritisch. |
| Passwort-Hash (Server) | `argon2` (Bindung an die Referenzimplementierung) | Krypto-Primitive werden nie selbst gebaut. |
| Argon2id im Browser | etablierte WASM-Bibliothek (Kandidat: `hash-wasm`), Auswahl in der Umsetzung nach Prüfung von Pflege und Größe | Web Crypto bietet kein Argon2id (GK 3.2). |
| Web Push (RFC 8291/8292) | `web-push` | Verschlüsselung der Nachrichten ist Krypto. |
| Mailversand (SMTP zum Relay) | `nodemailer` | SMTP mit TLS und Authentifizierung ist umfangreich. |
| Übersetzungen | `intl-messageformat` (ICU MessageFormat) | Plural- und Auswahlregeln aller Sprachen. |
| Symmetrische Krypto, Ed25519, HMAC, Zufall | eingebaut: `node:crypto` und Web Crypto | Keine Zusatzbibliothek nötig. |

**Selbst entwickelt** werden: Rechte, Plugin-Host, Sync-Engine, Migrationen, Jobs, Audit-Log, TOTP (RFC 6238 auf Basis von HMAC aus `node:crypto`; der Algorithmus ist kurz und mit den Testvektoren aus dem RFC prüfbar [K4]), Drosselung, Admin-Bereich, Oberfläche.

**Regeln für Abhängigkeiten:** Jede neue Abhängigkeit braucht eine kurze Begründung im Pull Request. Versionen sind im Lockfile fest. Dependabot meldet Updates als Pull Requests; `npm audit` läuft in CI. Pakete mit Installationsskripten werden nur zugelassen, wenn es nötig ist (z. B. die native `argon2`-Bindung).

---

## 3. Repository und Entwicklungsumgebung

### 3.1 Struktur

```
/core                 Server-Kern (Fastify-App, Dienste, Migrationen)
/web                  PWA-Grundgerüst (Shell, Service Worker, Offline-Speicher)
/packages/shared      Gemeinsame Typen, JSON-Schemata, Sync-Protokoll, i18n-Werkzeuge
/packages/plugin-sdk  Typen und Hilfen für Plugin-Autoren (Server und UI)
/plugins/demo-notes   Test-Plugin (nur Entwicklung und Tests)
/tools/signer         Signierwerkzeug (läuft nur auf Christians Rechner)
/dev                  Entwicklungsumgebung (Container-Definitionen, Testzertifikate-Skript)
/docs                 Konzepte, Entscheidungen
```

### 3.2 Lokale Umgebung

- PostgreSQL läuft in einem Container (Docker oder Podman), der Kern direkt mit Node.js. Ein Befehl startet alles (`npm run dev`).
- **HTTPS im Heimnetz** (GK 4): Service Worker, Web Push und Passkeys brauchen einen sicheren Kontext. Vorschlag (Entscheidung K-E2): eine eigene kleine Test-Zertifizierungsstelle, erzeugt per Skript mit OpenSSL. Ihr Wurzelzertifikat wird auf den Testgeräten installiert. Auf iOS muss man ihm danach unter Einstellungen → Allgemein → Info → Zertifikatsvertrauenseinstellungen ausdrücklich vertrauen [K5]. Alternative: eine echte Test-Subdomain mit Let's-Encrypt-Zertifikat (braucht eine eigene Domain).
- Hinweis: Passkeys sind an den Domainnamen gebunden (WebAuthn „Relying Party ID“ [GK Q2]). In der Testumgebung registrierte Passkeys gelten nicht für den späteren Produktivserver und umgekehrt.
- Testdaten: ein Skript legt eine Familie mit Admin, Erwachsenem, Kind und Gast an.

### 3.3 CI (GitHub Actions)

Bei jedem Pull Request: Lint, Formatprüfung, Typprüfung, Unit-Tests, Integrationstests gegen PostgreSQL (Service-Container), Ende-zu-Ende-Tests mit Playwright (Chromium und WebKit), Prüfung auf vollständige Übersetzungen, `npm audit`. Die Pipeline wird sparsam gehalten (Cache für Abhängigkeiten, Ende-zu-Ende-Tests nur bei Änderungen an `web`, `core`, `packages` (gemeinsame Schemas, Sync-Protokoll, Plugin-SDK) oder Plugins sowie an der CI-Konfiguration, den Abhängigkeiten (`package.json`, `package-lock.json` im Wurzelverzeichnis und in Unterprojekten) und gemeinsamen Build-Einstellungen (`tsconfig`, Vite-Konfiguration); Dependabot-Updates lösen damit immer auch Ende-zu-Ende-Tests aus), weil das kostenlose Kontingent an Actions-Minuten für private Repos begrenzt ist (GK 10).

---

## 4. Laufzeitarchitektur

- **Ein Prozess:** Der Kern ist ein einzelner Node.js-Prozess hinter nginx (E2). nginx terminiert TLS und spricht HTTP/2 mit den Browsern, der Kern lauscht nur auf `127.0.0.1`.
- **Module im Kern:** `auth`, `accounts`, `access` (Rechte), `plugins`, `sync`, `notify`, `jobs`, `audit`, `admin`, `i18n`, `config`, `crypto` (Feldverschlüsselung, Schlüsselverwaltung). Jedes Modul hat eine schmale, typisierte Schnittstelle; Module greifen nicht auf fremde Tabellen zu.
- **URL-Bereiche:**
  - `/` PWA (statische Dateien, vom Service Worker gecacht)
  - `/api/v1/...` JSON-API des Kerns (Sitzungs-Cookie)
  - `/api/v1/plugins/<id>/...` Routen der Plugins (Sitzungs-Cookie, Rechteprüfung durch den Kern)
  - `/plugins/<id>/ui/...` UI-Dateien der Plugins (statisch, signiert)
  - `/admin/...` Admin-Bereich (nur aus dem Heimnetz)
  - `/dav/...` reserviert für CalDAV in Phase 3 (App-Passwörter, keine Cookies)
- **Konfiguration:** eine Datei `config.toml` (nur für den Dienstnutzer lesbar) plus getrennte Schlüsseldateien (Abschnitt 11.3). Keine Geheimnisse in Umgebungsvariablen, Logs oder im Repo.

---

## 5. Datenmodell des Kerns

Alle Kern-Tabellen liegen im Schema `core`. IDs sind zufällige UUIDs (Version 4), keine fortlaufenden Nummern, damit IDs nichts über Anzahl oder Reihenfolge verraten. Zeitpunkte als `timestamptz` (UTC).

| Tabelle | Inhalt (Auszug) |
|---|---|
| `users` | Anzeigename, Sprache, Zeitzone, Rolle, Status (eingeladen, aktiv, deaktiviert), Flag „Offline erlaubt“ |
| `invitations` | Einladungstoken (nur Hash gespeichert), Ablauf, vorgesehene Rolle |
| `password_credentials` | Argon2id-Hash mit Parametern, Zeitpunkt der letzten Änderung |
| `totp_credentials` | TOTP-Geheimnis (feldverschlüsselt), letzter verwendeter Zeitschritt (gegen Wiederverwendung) |
| `webauthn_credentials` | Credential-ID, öffentlicher Schlüssel, Signaturzähler, Transporte, Name („iPhone von …“) |
| `recovery_codes` | Hash je Code, verwendet am |
| `recovery_links` | Hash des Tokens eines Wiederherstellungslinks vom Admin, Nutzer, ausgestellt von, Ablauf, verwendet am (Abschnitt 6.6) |
| `devices` | Gerät je Nutzer: Name, angelegt, zuletzt gesehen, Status, **Geräteschlüssel** (feldverschlüsselt, GK 3.4.3) |
| `sessions` | Hash des Sitzungstokens, Gerät, angelegt, zuletzt aktiv, Ablauf, Art (normal oder Wiederherstellung), Zeitpunkt der letzten starken Anmeldung |
| `login_throttle` | bestätigte Fehlversuche je Konto und je IP-Präfix (beim IP-Präfix mit Zeitpunkten für das 24-Stunden-Fenster), Zeitpunkt des zuletzt zugelassenen Versuchs |
| `login_inflight` | laufende Anmeldeversuche: Konto, IP-Präfix, Ablauf (Abschnitt 6.3) |
| `groups`, `group_members` | Gruppen und Mitgliedschaften |
| `grants` | Rechte: wer (Nutzer/Gruppe) darf was (Aktion) auf welchem Objekt oder Plugin |
| `plugins` | ID, Version, Status, Manifest, Hash des Pakets (SHA-256 von `FILES.sha256`), Signaturschlüssel-ID, bestätigte Rechte |
| `signing_keys` | Öffentliche Signaturschlüssel, Status (aktiv, zurückgezogen) |
| `sync_objects` | Objekt-ID, Typ, Besitzer-Plugin, aktuelle Revision, gelöscht (Grabstein), Inhalt (JSON, ggf. feldverschlüsselt) |
| `sync_revision` | eine einzige Zeile mit der zuletzt vergebenen Revision (Abschnitt 9.2) |
| `sync_changes` | fortlaufende Revision, Objekt, Änderungs-ID des Clients, Gerät, Zeitpunkt, Ergebnis (für die Wiederholung) |
| `sync_object_versions` | frühere Stände eines Objekts: Objekt, Revision, Inhalt (wie in `sync_objects` ggf. feldverschlüsselt), ersetzt am (Abschnitt 9.2, „Frühere Stände aufbewahren“) |
| `sync_conflicts` | Konflikteinträge: Objekt, Basisrevision und Vorwerte, abgelehnte Änderung (oder Löschabsicht), Status |
| `push_subscriptions` | Endpunkt und Schlüssel je Gerät |
| `notification_prefs` | Einstellungen je Nutzer, Plugin und Ereignis |
| `jobs` | geplante Aufgaben, nächster Lauf, Sperre, letzter Fehler |
| `audit_log` | nur anhängen: Zeitpunkt, Akteur, Aktion, Ziel, Ergebnis, IP-Präfix |
| `settings` | Kern-Einstellungen |

**Plugin-Daten:** Jedes Plugin bekommt ein eigenes Schema `plugin_<id>` und eine eigene Datenbankrolle (GK 3.3). Objekte, die synchronisiert werden, liegen nicht im Plugin-Schema, sondern in `core.sync_objects` und gehören dem Plugin; der Zugriff läuft nur über die Sync-Schnittstelle des Kerns. So gilt die Konfliktlogik für alle Plugins gleich. Plugins dürfen im eigenen Schema zusätzliche Tabellen anlegen (z. B. Indizes oder nicht synchronisierte Daten).

**Löschen:** Synchronisierte Objekte werden zuerst als Grabstein markiert, damit Offline-Geräte vom Löschen erfahren. Grabsteine werden nach der maximalen Offline-Dauer plus Puffer endgültig entfernt (Vorschlag: 60 Tage), **außer solange ein offener Konflikt auf das Objekt verweist**; dann bleiben Grabstein und benötigte frühere Stände bis zur Auflösung des Konflikts erhalten. Ein Gerät, das länger offline war, muss sich vollständig neu synchronisieren.

---

## 6. Konten und Anmeldung

### 6.1 Konten anlegen

- **Keine offene Registrierung.** Konten entstehen nur über eine Einladung durch einen Admin. Der erste Admin wird bei der Einrichtung per Kommandozeile auf dem Server angelegt.
- Die Einladung ist ein Link mit zufälligem Token (mindestens 128 Bit), einmalig nutzbar, 7 Tage gültig. Gespeichert wird nur ein Hash des Tokens.
- Beim Annehmen legt die Person einen Passkey an und/oder ein Passwort plus TOTP. Danach werden 10 Wiederherstellungscodes angezeigt (je 10 Zeichen aus dem Crockford-Base32-Alphabet: 32 Zeichen ohne die verwechselbaren Buchstaben I, L, O, U [K15]); gespeichert werden nur ihre Hashes.

### 6.2 Anmeldewege

| Weg | Ablauf |
|---|---|
| Passkey | WebAuthn mit Nutzerverifikation (`userVerification: required`). Eine Passkey-Anmeldung gilt als starke Anmeldung. |
| Passwort plus TOTP | Beides in einem Schritt prüfen; die Antwort verrät nicht, welcher Teil falsch war. TOTP nach RFC 6238 mit 30-Sekunden-Schritt und 6 Ziffern [K4], Toleranz ±1 Schritt, ein bereits verwendeter Zeitschritt wird abgelehnt. |
| Wiederherstellungscode | Ersetzt einmalig den verlorenen Faktor: bei Passwort plus TOTP das TOTP, bei Konten nur mit Passkey den Passkey. Ergebnis ist keine normale Sitzung, sondern eine eingeschränkte Wiederherstellungssitzung (Abschnitt 6.6). |

**Passwortregeln** (nach NIST SP 800-63B-4 [K6]):

- Mindestlänge 8 Zeichen für Passwörter, die nur zusammen mit TOTP benutzt werden. **Für Konten mit Offline-Zugriff mindestens 15 Zeichen**, weil das Passwort dort allein den Offline-Schlüssel schützt und damit wie ein einzelner Faktor wirkt (GK 3.4.3, „Grenze des Verfahrens“).
- Keine Vorgaben zu Zeichenarten (NIST schließt solche Regeln aus [K6]); stattdessen Abgleich mit einer mitgelieferten Liste häufiger und geleakter Passwörter.
- Lange Passphrasen sind erlaubt (bis 256 Zeichen); Einfügen aus Passwortmanagern ist erlaubt.

**Passwort-Hash auf dem Server:** Argon2id mit m = 64 MiB, t = 3, p = 4, 128-Bit-Salt (zweite Empfehlung aus RFC 9106 [GK Q4]). Das liegt über der Mindestempfehlung von OWASP (19 MiB, t = 2, p = 1 [K7]). Weil jede Prüfung 64 MiB Speicher braucht, laufen höchstens 4 Prüfungen gleichzeitig (zusammen 256 MiB). Weitere Anfragen warten in einer **begrenzten** Warteschlange: höchstens 32 Plätze für alle Nutzer zusammen und höchstens 5 Sekunden Wartezeit. Ist die Warteschlange voll oder die Wartezeit um, antwortet der Server sofort mit `503 Service Unavailable` und `Retry-After`, ohne das Passwort zu prüfen; das zählt nicht als Fehlversuch. Die Drosselung (6.3) und die Ratenbegrenzung (11.4) werden vor dem Einreihen geprüft, damit gedrosselte Versuche keinen Platz belegen. Ein wartender Eintrag hält nur die Anfrage selbst (höchstens 1 MiB nach 11.4, praktisch wenige KiB), so bleibt der Speicherbedarf auch bei einer verteilten Flut begrenzt. Passkey-Anmeldungen brauchen kein Argon2 und laufen an der Warteschlange vorbei. Die Grenzwerte sind Startwerte und werden mit einem Lasttest auf der Zielhardware überprüft. Die Parameter werden mit dem Hash gespeichert, damit sie später erhöht werden können (Neuberechnung bei der nächsten erfolgreichen Anmeldung).

### 6.3 Drosselung bei Fehlversuchen (E9)

- Zähler je Konto und je IP-Präfix (IPv4: /32, IPv6: /64, weil ein einzelner IPv6-Anschluss viele Adressen hat).
- **IP-Zähler verfällt:** Beim IP-Präfix zählen nur bestätigte Fehlversuche der letzten 24 Stunden (gleitendes Zeitfenster). Hinter einer IPv4-Adresse sitzen oft mehrere Menschen (z. B. alle Geräte eines Haushalts hinter dem Router); ohne Verfall würden alte Fehlversuche einer Person dauerhaft alle anderen ausbremsen. Anfragen aus dem Heimnetz (private Adressen, wie beim Admin-Bereich, 11.7) haben keinen IP-Zähler, dort gilt nur der Kontozähler. Der Kontozähler verfällt nicht; er wird nur durch eine erfolgreiche Anmeldung zurückgesetzt. Ein Test prüft, dass Fehlversuche einer Person nach 24 Stunden andere Nutzer am selben Anschluss nicht mehr bremsen.
- Die ersten 5 Fehlversuche sind frei. Danach muss vor jedem weiteren Versuch eine Wartezeit vergehen, die sich jeweils verdoppelt: 1 s, 2 s, 4 s … bis höchstens 1 Stunde. Ein Erfolg setzt den Kontozähler zurück.
- Die Wartezeit gilt auch für richtige Passwörter, damit sie nicht durch Ausprobieren umgangen werden kann.
- **Keine Umgehung durch parallele Anfragen:** Bestätigte Fehlversuche und laufende Versuche werden getrennt geführt.
  - `login_throttle` zählt je Konto und je IP-Präfix nur **bestätigte** Fehlversuche (Argon2 ist gelaufen, Ergebnis falsch) und speichert den Zeitpunkt des **zuletzt zugelassenen** Versuchs. Die Wartezeit läuft ab diesem Zeitpunkt.
  - `login_inflight` enthält je laufendem Versuch eine Zeile (Konto, IP-Präfix, Ablauf nach 30 Sekunden). Abgelaufene Zeilen zählen nicht mehr; so bleibt nach einem Absturz nichts dauerhaft hängen.
  - **Reservieren:** In einer Transaktion werden die beiden `login_throttle`-Zeilen gesperrt (`SELECT … FOR UPDATE`, fehlende Zeilen vorher per `INSERT … ON CONFLICT DO NOTHING` angelegt). Die Wartezeit wird aus *bestätigte Fehlversuche + laufende Versuche* berechnet, für Konto und IP-Präfix getrennt; ist eine der beiden Wartezeiten nicht abgelaufen, endet die Transaktion ohne Änderung, Argon2 läuft nicht, und der Client muss warten. Sonst wird eine `login_inflight`-Zeile angelegt und in **beiden** `login_throttle`-Zeilen der Zeitpunkt des zuletzt zugelassenen Versuchs auf jetzt gesetzt (ohne die bestätigten Fehlversuche zu erhöhen). Ein parallel ankommender Versuch sieht damit nach dem Warten auf die Sperre den neuen Zeitpunkt und den laufenden Versuch und muss die nächste Wartezeit abwarten. Alles oder nichts: Beide Prüfungen und die Reservierung geschehen unter denselben Sperren.
  - **Abschließen:** Nach Argon2 wird in einer Transaktion die eigene `login_inflight`-Zeile gelöscht und das Ergebnis eingetragen: falsch → bestätigter Fehlversuch +1 bei beiden Zählern; richtig → Kontozähler auf 0, IP-Zähler unverändert. Ein Reset löscht nur bestätigte Fehlversuche; laufende Versuche anderer Anfragen sind davon getrennt und werden nach ihrem Abschluss ganz normal gezählt.
  - **Abweisen ohne Prüfung** (Warteschlange voll oder Wartezeit abgelaufen, `503`): Nur die eigene `login_inflight`-Zeile wird gelöscht; der Zeitpunkt des zugelassenen Versuchs bleibt stehen (sonst könnte man die Wartezeit über Überlast-Abweisungen umgehen), es gibt nichts zurückzurechnen, und Fehlversuche anderer Anfragen bleiben unberührt. Das entspricht der Regel „zählt nicht als Fehlversuch“ aus 6.2.
  - **Alarme** (unten) zählen nur bestätigte Fehlversuche.
  - **Tests:** parallele Anmeldeversuche, parallele richtige und falsche Versuche am selben Konto, viele erfolgreiche Anmeldungen hintereinander vom selben IP-Präfix, ein paralleler Schwall von Versuchen direkt nach Ablauf der maximalen Wartezeit, Abweisung wegen voller Warteschlange und Absturz während eines laufenden Versuchs.
- Rechnerisch sind so in der ersten Stunde knapp 20 Versuche möglich, danach höchstens einer pro Stunde. 100 aufeinanderfolgende Fehlversuche, die von NIST genannte Obergrenze [K6], würden damit mehr als drei Tage dauern.
- **Kein Aussperren durch Angreifer:** Die Wartezeit des Kontozählers gilt nur für die Anmeldung mit Passwort. Mit einem Passkey kann sich der Nutzer jederzeit anmelden, auch wenn ein Angreifer gerade sein Passwort durchprobiert; das ist der Grund für E9.
- **Bekannte Geräte (Device Cookies nach OWASP [K17]):** Nach jeder erfolgreichen Passwort-Anmeldung bekommt der Browser ein Geräte-Cookie (`__Host-device`, `Secure`, `HttpOnly`) mit Kontokennung, Zufallswert und HMAC-Signatur, das bei jeder Anmeldung erneuert wird. Versuche mit gültigem Geräte-Cookie für dieses Konto werden **je Cookie** gedrosselt, nicht über den Kontozähler. Der Kontozähler gilt nur für Versuche ohne gültiges Geräte-Cookie. Ein Angreifer, der das Passwort von außen durchprobiert, bremst damit nur unbekannte Geräte; der Nutzer kann sich auf seinen bekannten Geräten weiter anmelden, auch ohne Passkey. Für unbekannte Geräte bleibt der Passkey der Weg ohne Wartezeit (E9). Ein Test simuliert einen Angriff auf ein Konto und prüft, dass ein bekanntes Gerät sich weiter anmelden kann.
- Ab 10 Fehlversuchen in Folge geht ein Alarm an die Admins (Push und Mail), ab 20 zusätzlich an den betroffenen Nutzer.
- Passkey-Anmeldungen sind nicht erratbar und werden nicht gedrosselt, wohl aber in der Zahl der Anfragen je IP begrenzt (allgemeine Ratenbegrenzung, Abschnitt 11.4).

### 6.4 Sitzungen und Geräte

- Ein Sitzungstoken hat 256 Bit Zufall, liegt im Cookie `__Host-session` (`Secure`, `HttpOnly`, `SameSite=Lax`, `Path=/`). In der Datenbank steht nur sein Hash.
- **Geräte:** Bei der ersten Anmeldung in einem Browser wird ein Gerät angelegt (mit Namen, z. B. „Christians Pixel“). Sitzungen gehören zu einem Gerät. Der Geräteschlüssel für den Offline-Speicher (GK 3.4.3, E14) gehört ebenfalls zum Gerät.
- **Laufzeit:** Sitzungen enden nach 30 Tagen ohne Nutzung, spätestens nach 90 Tagen. Danach ist eine neue Anmeldung nötig. (Die PWA bleibt offline nutzbar, solange die maximale Offline-Dauer von 30 Tagen nicht abgelaufen ist, E10.)
- **Starke Anmeldung für heikle Aktionen:** Faktoren ändern, Passwort ändern, Geräte entfernen, Admin-Aktionen und Plugin-Installation verlangen eine starke Anmeldung, die höchstens 10 Minuten alt ist; sonst fragt die Oberfläche erneut nach Passkey oder Passwort plus TOTP. Einzige Ausnahme ist die Wiederherstellungssitzung (Abschnitt 6.6): Sie erlaubt ohne den verlorenen Faktor genau das Einrichten eines Ersatzfaktors, sonst nichts.
- **Fernabmeldung:** Jeder Nutzer sieht seine Geräte und Sitzungen und kann sie einzeln oder alle beenden. Ein entferntes Gerät verliert sofort seinen Geräteschlüssel auf dem Server (GK 3.4.3, Schritt 7). Admins können das für alle Konten.

### 6.5 Schutz gegen gefälschte Anfragen (CSRF)

- `SameSite=Lax` allein reicht nach OWASP nicht als einzige Maßnahme [K8]. Zusätzlich prüft der Kern bei jeder ändernden Anfrage (`POST`, `PUT`, `PATCH`, `DELETE`) den Header `Sec-Fetch-Site` und lehnt alles außer `same-origin` ab. Fehlt der Header, wird ersatzweise `Origin` geprüft; fehlt auch dieser oder ist er `null`, wird die Anfrage abgelehnt [K8].
- Die API nimmt nur `Content-Type: application/json` an.

### 6.6 Kontowiederherstellung

Ziel: Wer einen Faktor verliert, kommt wieder in sein Konto, ohne dass ein einzelner Wiederherstellungscode mehr Macht hat als nötig.

- **Wiederherstellungssitzung:** Ein gültiger Wiederherstellungscode (bei Passwort plus TOTP zusammen mit dem Passwort, bei Konten nur mit Passkey allein) erzeugt eine eigene Sitzungsart. Sie gilt höchstens 10 Minuten, wird nicht verlängert und erlaubt nur: einen **zusätzlichen** Passkey einrichten, das TOTP ersetzen (nur bei Passwort plus TOTP, wo zusätzlich das Passwort nötig war) oder bei Konten nur mit Passkey zusätzlich Passwort plus TOTP einrichten, sowie eigene Geräte und Sitzungen **ansehen**. Bestehende Passkeys entfernen, Sitzungen oder Geräte beenden und neue Wiederherstellungscodes erzeugen geht nur in einer normalen Sitzung mit starker Anmeldung. So kann ein Angreifer mit gestohlenem Code den echten Nutzer nicht aussperren: Dessen Passkeys und Sitzungen bleiben, er wird benachrichtigt (unten) und kann den fremden Faktor in einer normalen Sitzung entfernen. Alle übrigen Endpunkte lehnen sie ab, auch das Lesen von Daten. Nach dem Einrichten des Ersatzfaktors endet sie; der Nutzer meldet sich mit dem neuen Faktor normal an.
- **Drosselung:** Wiederherstellungscodes zählen wie Passwortversuche (Abschnitt 6.3, je Konto und je IP-Präfix). Ein Code hat 10 Zeichen aus 32 möglichen, also 32¹⁰ = 2⁵⁰ Möglichkeiten (50 Bit); zusammen mit der Drosselung ist Erraten aussichtslos.
- **Benachrichtigung:** Jede Nutzung eines Codes und jeder in der Wiederherstellungssitzung eingerichtete Faktor wird ins Audit-Log geschrieben und per Push und Mail an den Nutzer und die Admins gemeldet. So fällt ein gestohlener Code sofort auf.
- **Restbestand:** Nach der Nutzung zeigt die App, wie viele Codes übrig sind, und bietet an, einen neuen Satz zu erzeugen (der alte wird dann ungültig). Das erfordert eine normale starke Anmeldung.
- **Hilfe durch einen Admin:** Hat jemand weder Faktor noch Code (oder das Passwort vergessen), stellt ein Admin nach eigener starker Anmeldung einen Wiederherstellungslink aus: einmalig, 24 Stunden gültig, nur der Hash wird gespeichert, Übergabe persönlich oder über einen vertrauenswürdigen Weg. Der Link öffnet ebenfalls nur eine Wiederherstellungssitzung. Für den letzten verbliebenen Admin geht das per Kommandozeile auf dem Server.
- **Empfehlung in der Oberfläche:** Konten nur mit Passkey werden gebeten, mindestens zwei Passkeys auf verschiedenen Geräten anzulegen.
- **Offline-Daten:** Ein neues Passwort wird für den Offline-Speicher wie eine normale Passwortänderung behandelt (Neuverpacken über den Geräteschlüssel, GK 3.4.3).

---

## 7. Gruppen, Rollen und Rechte

### 7.1 Rollen

| Rolle | Darf grundsätzlich |
|---|---|
| Admin | Alles, einschließlich Nutzer, Plugins, Einstellungen (Admin-Bereich nur aus dem Heimnetz) |
| Erwachsener | Alle Plugins nutzen, eigene Objekte anlegen und teilen |
| Kind | Plugins nutzen, die der Admin für Kinder freigibt; teilen nur innerhalb der Familie; keine externen Konten verbinden |
| Gast | Nur ausdrücklich geteilte Objekte sehen oder bearbeiten |

Ein Nutzer hat genau eine Rolle. Die Rolle legt fest, was jemand höchstens darf; konkrete Rechte kommen über Freigaben dazu.

### 7.2 Freigaben

- Ein **Objekt** (z. B. eine Einkaufsliste, später ein Kalender) hat einen Besitzer. Der Besitzer kann es für Nutzer oder Gruppen freigeben, mit den Stufen `view`, `edit`, `manage` (darf weiter freigeben).
- **Private Objekte** sind nur für den Besitzer sichtbar, auch nicht für Admins in der Oberfläche. Ehrliche Grenze: Wer den Server und die Datenbankschlüssel kontrolliert, kann technisch alles lesen, und der Datenexport beim Entfernen eines Plugins (8.5) enthält auch private Objekte. Die Regel schützt vor neugierigen Familienmitgliedern, nicht vor dem Betreiber selbst.
- Plugins legen fest, welche Objekttypen es gibt und ob Unterobjekte (z. B. Artikel einer Liste) die Freigabe ihres Elternobjekts erben. Vorschlag: Vererbung ist der Standard.

### 7.3 Prüfung

- Eine einzige Funktion im Kern entscheidet: `can(user, action, resource) → allow | deny`. Jede Route und jede Sync-Operation ruft sie auf, bevor Daten gelesen oder geändert werden. Standard ist `deny`.
- Plugins können eigene Aktionen definieren, aber die Prüfung nicht umgehen: Ihre Routen bekommen vom Kern nur Daten, für die die Prüfung erfolgreich war.
- Für die Prüfung gibt es eine Tabelle von Testfällen (Rolle × Aktion × Freigabe), die vollständig automatisch getestet wird.

---

## 8. Plugin-System

### 8.1 Paketformat

Ein Plugin-Paket ist ein Archiv (`.fap`, intern ein tar-Archiv) mit:

```
manifest.json      Metadaten und Rechte
server/index.js    Server-Teil (gebündeltes ES-Modul)
ui/                UI-Teil (gebündelte ES-Module, CSS, Bilder)
migrations/        SQL-Migrationen für das eigene Schema
i18n/de.json, en.json
FILES.sha256       SHA-256 jeder Datei außer FILES.sha256 und SIGNATURE
SIGNATURE          Ed25519-Signatur über FILES.sha256, plus ID des Signaturschlüssels
```

**Reihenfolge ohne Zirkelbezug:** `FILES.sha256` listet alle Dateien des Pakets außer sich selbst und `SIGNATURE` (eine Zeile je Datei: Hash in Hex, Leerzeichen, relativer Pfad mit `/`, UTF-8, nach Pfad sortiert, Zeilenende `\n`). `SIGNATURE` signiert genau die Bytes von `FILES.sha256`. Damit deckt die Signatur über `FILES.sha256` indirekt jede andere Datei ab, auch `manifest.json`. Das Signierwerkzeug schreibt zuerst `FILES.sha256`, dann `SIGNATURE`; der Kern prüft in umgekehrter Reihenfolge (erst Signatur, dann Hashes). „Keine zusätzlichen Dateien“ in 8.2 heißt: Außer den in `FILES.sha256` gelisteten Dateien gibt es nur genau diese beiden.

**Manifest (Beispiel):**

```json
{
  "id": "shopping",
  "version": "1.0.0",
  "coreApi": "^1.0.0",
  "name": { "de": "Einkaufslisten", "en": "Shopping lists" },
  "permissions": ["storage:own", "sync", "notify", "jobs"],
  "network": [],
  "objectTypes": ["list", "item"],
  "roles": { "child": true, "guest": false }
}
```

### 8.2 Prüfung und Installation

1. Admin lädt das Paket im Admin-Bereich hoch (nur Heimnetz, starke Anmeldung).
2. Der Kern prüft: Signatur gültig und Schlüssel aktiv, jede Datei passt zu `FILES.sha256`, keine zusätzlichen Dateien (außer `FILES.sha256` und `SIGNATURE`), nur normale Dateien und Verzeichnisse (keine symbolischen oder harten Links, keine Gerätedateien), Pfade ohne `..` und ohne absolute Pfade, Größenlimit beim Entpacken, `coreApi` passt zur Kernversion.
3. Der Admin sieht die verlangten Rechte und bestätigt sie.
4. Der Kern legt Schema und Datenbankrolle an, führt die Migrationen in einer Transaktion aus und aktiviert das Plugin.
5. **Bei jedem Start** wiederholt der Kern vor dem Laden jedes Plugins die vollständige Prüfung aus Schritt 2, soweit sie die installierten Dateien betrifft: Die Ed25519-Signatur über `FILES.sha256` muss gültig sein, der Signaturschlüssel muss in `signing_keys` noch aktiv sein und die ID aus `plugins` tragen, jede Datei muss zu ihrem Hash passen, und es dürfen keine zusätzlichen Dateien oder Links vorhanden sein. Außerdem muss das Paket **genau das installierte** sein: Der SHA-256 von `FILES.sha256` sowie Plugin-ID und Version aus `manifest.json` müssen mit den Werten in `plugins` übereinstimmen. So kann niemand ein älteres, ebenfalls gültig signiertes Paket (z. B. mit einer inzwischen behobenen Lücke oder einem älteren Schema) unterschieben. Ein Wechsel auf eine andere Version, auch zurück, geht nur über den Installationsweg im Admin-Bereich. Ein reiner Hash-Vergleich würde nicht reichen, weil ein Angreifer mit Schreibzugriff sonst eine Datei und ihren Eintrag in `FILES.sha256` gemeinsam austauschen könnte. Schlägt etwas fehl, wird das Plugin nicht geladen, und ein Alarm geht raus.
6. **Schlüssel zurückziehen:** Wird ein Signaturschlüssel im laufenden Betrieb zurückgezogen, deaktiviert der Kern sofort alle damit signierten Plugins (ohne ihre Daten zu löschen) und meldet das den Admins.

**Signierwerkzeug** (`/tools/signer`): läuft auf Christians Rechner, baut das Plugin aus einem sauberen Git-Stand (ohne lokale Änderungen), schreibt `FILES.sha256` und signiert mit dem privaten Ed25519-Schlüssel. Der private Schlüssel liegt verschlüsselt (Passphrase) auf dem Rechner und zusätzlich offline gesichert (GK 3.3). Schlüsselwechsel: Ein neuer öffentlicher Schlüssel wird im Admin-Bereich hinzugefügt; der alte kann zurückgezogen werden, danach laden mit ihm signierte Pakete nicht mehr.

**Entwicklungsmodus:** Nur wenn der Kern mit `NODE_ENV=development` und zusätzlich der Einstellung `plugins.allowUnsigned = true` läuft, lädt er unsignierte Plugins aus dem Repo. Ein Produktions-Build enthält diesen Code-Pfad nicht. Ein automatischer Test prüft, dass ein Produktions-Build unsignierte Plugins ablehnt.

### 8.3 Laufzeit-Schnittstelle

Der Server-Teil exportiert eine Funktion `register(ctx)`. Über `ctx` bekommt das Plugin **nur** die Dienste, die sein Manifest verlangt und der Admin bestätigt hat:

| Dienst | Recht | Inhalt |
|---|---|---|
| `ctx.routes` | immer | Routen unter `/api/v1/plugins/<id>/`, mit JSON-Schema-Validierung |
| `ctx.db` | `storage:own` | Datenbankverbindung mit der Rolle des Plugins (nur eigenes Schema) |
| `ctx.sync` | `sync` | Objekttypen registrieren, Konflikteinheiten und Prüfregeln festlegen (Abschnitt 9); Sync-Objekte lesen und ändern, aber nur **im Namen eines Nutzers** (aus der aktuellen Anfrage), mit automatischer Rechteprüfung `can(...)` im Kern und denselben Regeln wie der Sync-Endpunkt |
| `ctx.access` | immer | `can(...)` für eigene Objekte, Freigaben verwalten |
| `ctx.notify` | `notify` | Benachrichtigungen an Nutzer senden |
| `ctx.jobs` | `jobs` | zeitgesteuerte Aufgaben |
| `ctx.http` | `network` nicht leer | ausgehende HTTP-Anfragen, nur zu den im Manifest genannten Hosts |
| `ctx.log`, `ctx.i18n`, `ctx.config` | immer | Logging mit Plugin-Kennung, Übersetzungen, eigene Einstellungen |

Wie im Gesamtkonzept beschrieben, ist das eine Schutzschicht gegen Fehler, keine Sicherheitsgrenze: Ein Plugin im selben Prozess könnte z. B. `node:fs` direkt importieren (GK 3.3). Zusätzlich gilt daher:

- Ein Lint-Regelsatz für Plugins verbietet direkte Importe von `node:fs`, `node:net`, `node:child_process`, `pg` usw.; CI bricht bei Verstößen ab.
- Plugins werden nur von Christian signiert, nachdem der Code im Repo reviewt wurde.
- **Rechte auf Plugin-Daten:** Jede Plugin-Route bekommt den angemeldeten Nutzer aus der Anfrage; ein Aufruf ohne Nutzer ist nur für Jobs möglich und läuft dann als Plugin selbst, ohne Zugriff auf Sync-Objekte anderer Plugins. Für Daten, die ein Plugin im eigenen Schema hält (nicht synchronisiert), ist das Plugin selbst für die Prüfung mit `ctx.access.can(...)` zuständig; das Plugin-SDK bietet dafür Helfer, und die Plugin-Vorlage enthält Pflicht-Tests, die Zugriffe ohne Recht (anderer Nutzer, Kind, Gast) ablehnen müssen. Der Review vor dem Signieren prüft das ausdrücklich.

### 8.4 UI-Teil

- Die PWA lädt den UI-Teil eines aktiven Plugins per dynamischem `import()` von `/plugins/<id>/ui/`. Die Content Security Policy erlaubt Skripte nur vom eigenen Ursprung.
- Das Plugin bekommt ein UI-Kontextobjekt mit Navigation, i18n, Zugriff auf den verschlüsselten Offline-Speicher (nur eigene Objekttypen) und den Sync-Client.
- Plugins können Seiten (eigene Route in der App) und Widgets (Kacheln auf der Startseite) registrieren.

### 8.5 Lebenszyklus

`hochgeladen → installiert → aktiv ⇄ deaktiviert → entfernt`

- **Update:** neues signiertes Paket, Migrationen laufen in einer Transaktion; schlägt eine fehl, bleibt die alte Version aktiv.
- **Deaktivieren:** Routen und Jobs werden abgeschaltet, Daten bleiben.
- **Entfernen:** Admin wählt Export (JSON-Archiv), Aufbewahren oder Löschen (GK 3.3). Löschen entfernt Schema, Rolle, Sync-Objekte und Freigaben des Plugins; das Audit-Log bleibt.
- Abhängigkeiten zwischen Plugins sind in Phase 1 nicht vorgesehen (schlank zuerst); die Schnittstelle wird erst gebaut, wenn ein Plugin sie braucht.

---

## 9. Sync-Engine

Umsetzung von GK 3.5.

### 9.1 Protokoll

| Endpunkt | Zweck |
|---|---|
| `GET /api/v1/sync/changes?since=<rev>&types=...` | Alle für den Nutzer sichtbaren Änderungen seit Revision `rev`, seitenweise. Weil eine einzige Revision (z. B. eine entzogene Gruppenfreigabe) viele Objekte betreffen kann, ist der Cursor ein Paar aus Revision und Objekt-ID (`since=<rev>:<objekt-id>`), sortiert nach beiden. Eine Seite kann so mitten in einer Revision enden, ohne dass Einträge doppelt oder gar nicht geliefert werden. Der Client übernimmt den Cursor erst, wenn die Seite vollständig gespeichert ist. Enthält auch Objekte, die er neu sehen darf, und Hinweise auf Objekte, die er nicht mehr sehen darf (damit das Gerät sie löscht). Dafür erzeugen auch Änderungen an Freigaben eine Revision; jeder Eintrag trägt die Revision seines Ereignisses als Beobachtungsrevision (9.2). |
| `POST /api/v1/sync/push` | Liste von Änderungen: Änderungs-ID (UUID vom Gerät), Objekt-ID, Typ, Basisrevision, Operation (`create`, `update` mit geänderten Feldern, `delete`, oder plugin-eigene Operation) und **Vorwerte**: für jede berührte Konflikteinheit der Wert, von dem das Gerät beim Bearbeiten ausging (bei `delete` der ganze zuletzt gesehene Inhalt). Antwort je Änderung: Ergebnis (`applied`, `merged`, `conflict` oder `rejected` wegen Rechten oder Validierung) und **Ergebnisrevision** (die Revision, die diese Änderung erzeugt hat; bei `conflict`/`rejected` keine). Darf der Nutzer das Objekt **im Moment der Antwort** lesen, enthält die Antwort zusätzlich den aktuellen Serverstand als zusammengehöriges Paar aus **aktueller Objektrevision und Inhalt**; sonst nur Ergebnis und Grund, und das Gerät löscht seine Kopie. Beides trägt eine Beobachtungsrevision (9.2). |
| `GET /api/v1/sync/conflicts`, `POST .../resolve` | Konflikteinträge anzeigen und auflösen |
| `GET /api/v1/events` | Live-Aktualisierung per Server-Sent Events, solange die App offen ist. EventSource ist in allen gängigen Browsern verfügbar [K9]; über HTTP/2 entfällt die Grenze von 6 Verbindungen je Domain [K9]. Die Ereignisse enthalten nur „es gibt Neues ab Revision X“, die Daten holt der Client über `changes`. |

### 9.2 Regeln

- **Idempotenz:** Eine Änderungs-ID wird nur einmal angewendet; ein erneutes Senden liefert dasselbe Ergebnis mit derselben Ergebnisrevision. Gespeichert wird nur dieses Ergebnis, nicht der Objektinhalt. Den Serverstand für die Antwort liest der Server jedes Mal neu, immer zusammen mit der dazu passenden aktuellen Objektrevision (die nach Änderungen anderer Geräte höher als die Ergebnisrevision sein kann), und prüft dabei erneut das Leserecht, damit eine inzwischen entzogene Freigabe nicht über eine Wiederholung umgangen wird.
- **Revisionen:** Eine globale, fortlaufende Revision für alle Änderungen (auch an Freigaben). Jede Änderung läuft in einer eigenen Transaktion: Revisionszähler nehmen (siehe unten), Objekt sperren (`SELECT … FOR UPDATE`), Rechte prüfen, zusammenführen, prüfen, speichern, abschließen. Wird die Änderung nicht angewendet (`conflict`, `rejected`), schreibt die Transaktion nur den Konflikteintrag bzw. das Ergebnis für die Idempotenz; die dabei vergebene Revision bleibt ohne Objektänderung.
  - **Keine PostgreSQL-Sequenz:** `nextval` wird laut Dokumentation nie zurückgerollt, die Nummer ist also sofort beim Aufruf vergeben und nicht erst beim Abschluss der Transaktion [K13]. Daraus folgt (eigene Schlussfolgerung, die Dokumentation nennt die Commit-Reihenfolge nicht ausdrücklich): Zwei parallele Transaktionen können Revision 101 und 102 bekommen und in umgekehrter Reihenfolge abschließen. Ein Gerät, das nach dem Abschluss von 102 schon `since=102` abfragt, würde 101 nie erhalten. Der Umsetzungsschritt enthält einen Test, der genau diesen Fall mit zwei Verbindungen nachstellt.
  - **Stattdessen:** Die Revision kommt aus einer Zähler-Tabelle mit genau einer Zeile (`UPDATE core.sync_revision SET rev = rev + 1 RETURNING rev`). Die Zeilensperre hält bis zum Ende der Transaktion; eine zweite schreibende Transaktion wartet bis zum Commit der ersten und wendet ihr UPDATE dann auf den neuen Stand der Zeile an (Read Committed [K14]). Damit werden Revisionen in derselben Reihenfolge sichtbar, in der sie vergeben werden. Revisionen ohne sichtbare Änderung (z. B. bei `conflict`) sind möglich und unschädlich, weil der Cursor nur „größer als“ vergleicht. Der Zähler wird als **erster** Schritt genommen. Dadurch laufen alle schreibenden Sync-Transaktionen nacheinander, und die Rechteprüfung sieht immer alle zuvor abgeschlossenen Rechteänderungen. Das ist nötig, weil **jede Änderung, die Rechte beeinflusst** (Freigaben, Gruppenmitgliedschaften, Rollen, Kontosperrung, Aktivieren oder Entfernen eines Plugins), ebenfalls über diesen Zähler läuft und eine Revision erzeugt. Ohne diese Reihenfolge könnte eine Schreibtransaktion das Recht prüfen, während parallel eine geerbte Berechtigung (z. B. über eine Gruppe) entzogen wird, und danach trotzdem schreiben. Die Sperre wird damit länger gehalten (Rechteprüfung, Abgleich und Prüfregel, typischerweise Millisekunden); für die Schreiblast einer Familie ist diese Serialisierung unkritisch, ein Lasttest in der Umsetzung prüft das. Ein Test entzieht eine geerbte Berechtigung parallel zu einer Schreibanfrage.
- **Zusammenführen:** Drei-Wege-Abgleich je Konflikteinheit mit den Vorwerten des Geräts als Basis: Ist der aktuelle Serverwert gleich dem Vorwert, wird der neue Wert übernommen. Ist er gleich dem neuen Wert, ist nichts zu tun. Sonst hat jemand anderes dieselbe Einheit geändert, und die Änderung wird zum Konflikt. Bei `delete` wird nur gelöscht, wenn der Serverinhalt noch dem zuletzt gesehenen Inhalt entspricht. Weil die Basis mit der Änderung mitkommt, hängt der Abgleich nicht davon ab, welche Revision das Gerät zuletzt kannte oder ob der Server alte Stände noch hat. Plugins melden für jeden Objekttyp an:
  - **Konflikteinheiten:** Felder, die nur gemeinsam übernommen werden (GK 3.5).
  - **Prüfregel:** Funktion, die ein Ergebnis auf Gültigkeit prüft. Schlägt sie fehl, wird die Änderung zum Konflikt.
  - **Operationen** (optional): eigene, gut zusammenführbare Operationen, z. B. „Artikel abhaken“.
- **Mehrere Änderungen am selben Objekt auf einem Gerät:** Im Client gilt je Objekt: höchstens **eine gesendete** Änderung (unterwegs, noch ohne Antwort) und höchstens **eine wartende** Änderung. Weitere Bearbeitungen werden in die wartende Änderung eingearbeitet; je Konflikteinheit bleibt dabei der **erste** Vorwert erhalten (der Wert, von dem die Bearbeitung ursprünglich ausging, einschließlich eigener noch unbestätigter Änderungen). Eine gesendete Änderung wird bei Wiederholungen unverändert mit derselben Änderungs-ID geschickt (Idempotenz). Ein Push enthält daher nie zwei Änderungen am selben Objekt; der Server lehnt einen solchen Push als fehlerhaft ab.
  - **Nach `applied` oder `merged`:** Die wartende Änderung wird **unverändert mit ihren Vorwerten** gesendet; sie wird nicht auf den Serverstand umgeschrieben. Beispiel: Das Gerät ändert offline erst den Titel und dann den Preis von 10 auf 11, ein anderes Gerät hat den Preis inzwischen auf 12 gesetzt. Die Preisänderung kommt mit Vorwert 10 beim Server an, der Server hat 12, also entsteht ein Konflikt statt eines stillen Überschreibens. Ändert dasselbe Gerät zweimal den Titel (A → B, dann B → C), hat die zweite Änderung den Vorwert B; der Server hat nach der ersten Änderung ebenfalls B und übernimmt C ohne Konflikt.
  - **Nach `conflict`:** Die wartende Änderung für dieses Objekt wird angehalten und erst gesendet, wenn der Nutzer den Konflikt aufgelöst hat. So entstehen keine Folgekonflikte gegen einen Stand, den der Nutzer noch nicht gesehen hat.
  - **Nach `rejected`:** Darf der Nutzer das Objekt nicht mehr lesen, wird auch die wartende Änderung verworfen und der Nutzer sieht einen Hinweis. Darf er es noch lesen, aber nicht mehr bearbeiten (z. B. Freigabe von „bearbeiten“ auf „ansehen“ geändert), übernimmt das Gerät den Serverstand, zeigt die Ansicht nur noch lesend und bietet die nicht gespeicherten Änderungen (gesendete und wartende) einmalig zum Kopieren an; danach werden sie verworfen. Bei einer Ablehnung wegen Validierung wird die wartende Änderung angehalten und dem Nutzer angezeigt.
  - **Rechte im Gerät:** Jede Auskunft über ein Objekt (aus `changes` und aus Push-Antworten) enthält die wirksamen Rechte des Nutzers (`view` oder `edit`). Eine Rechteänderung erzeugt eine Revision (siehe „Revisionen“) und damit einen `changes`-Eintrag. So sperrt die App die Bearbeitung meist schon, bevor ein Push abgelehnt wird; offline vorgenommene Änderungen werden im Fall oben behandelt.
  - **Lokale Basis:** Als neue lokale Basis übernimmt das Gerät immer das Paar aus aktueller Objektrevision und Inhalt aus der Antwort, nie die Ergebnisrevision mit einem anderen Inhalt. Dabei gilt je Objekt eine **Beobachtungsrevision**: Jede Auskunft des Servers über ein Objekt (Inhalt oder Hinweis „nicht mehr sichtbar“, aus einer Push-Antwort oder aus `changes`) trägt die globale Revision, zu der sie gilt. Bei `changes` ist das die Revision des Ereignisses, also auch einer Freigabeänderung, nicht die Inhaltsrevision des Objekts; bei Push-Antworten die globale Revision des Datenbank-Schnappschusses, aus dem Inhalt und Leserecht gelesen wurden. Inhalt, Leserecht und Beobachtungsrevision müssen deshalb aus **einem** konsistenten Schnappschuss stammen: Bei einer neu angewendeten Änderung liest der Server sie in der schreibenden Transaktion erst **nach** dem Nehmen des Revisionszählers; weil der Zähler bis zum Commit gesperrt ist, kann dazwischen keine andere Revision abschließen, und Beobachtungsrevision ist die Ergebnisrevision. Bei einer Wiederholung (oder `rejected` ohne Schreibvorgang) liest der Server in einer eigenen Nur-Lese-Transaktion mit `REPEATABLE READ`, in der alle Abfragen denselben Schnappschuss sehen [K14]; die Beobachtungsrevision ist der Zählerstand in diesem Schnappschuss. Weil Revisionen in Vergabereihenfolge abschließen (siehe oben), enthält dieser Schnappschuss genau alle Änderungen bis zu diesem Zählerstand. Ein Test lässt Inhalts- und Freigabeänderungen zwischen den einzelnen Lesezugriffen abschließen und prüft, dass keine Auskunft mit zu hoher Revision entsteht. Das Gerät übernimmt eine Auskunft nur, wenn ihre Beobachtungsrevision **höher** ist als die zuletzt für dieses Objekt gespeicherte; sonst wird sie ignoriert. Die Beobachtungsrevision bleibt auch nach dem lokalen Löschen eines Objekts als kleiner Eintrag erhalten (bis zur vollständigen Neusynchronisation). Beispiel: Ein Objekt (Inhaltsrevision 100) wird bei Revision 200 entzogen und bei 201 wieder freigegeben. `changes` liefert das Objekt mit Beobachtungsrevision 201; ein verspäteter Hinweis „nicht mehr sichtbar“ mit 200 wird ignoriert. Ebenso kann eine verspätete Push-Antwort einen neueren Stand nicht zurücksetzen. Der `changes`-Cursor des Geräts wird davon getrennt geführt und nur durch `changes`-Antworten weitergesetzt. Tests decken verspätete Push-Antworten sowie Entzug und erneute Freigabe in vertauschter Reihenfolge ab.
  - **Tests:** Abgedeckt werden mindestens: mehrfaches Ändern desselben Objekts offline, Abbruch der Verbindung mitten im Push mit Wiederholung, gleichzeitige Änderung derselben Einheit auf einem anderen Gerät, und Wiederholung einer Antwort, nachdem ein anderes Gerät das Objekt geändert hat.
- **Konflikt:** Serverstand bleibt, die Client-Änderung wird als Konflikteintrag gespeichert, der Nutzer sieht ihn in der App (GK 3.5).
- **Ändern gegen Löschen:** Wurde ein Objekt auf dem Server gelöscht und auf einem Gerät geändert, wird die Änderung zum Konflikteintrag („gelöscht von …, deine Änderung wiederherstellen?“). Wurde es auf dem Gerät gelöscht und auf dem Server geändert, ebenfalls Konflikt („geändert von …, trotzdem löschen?“).
- **Frühere Stände aufbewahren:** Das Gerät speichert für jedes Objekt den zuletzt bestätigten Serverstand als Basis. Der Server legt bei jeder Änderung den bisherigen Stand in `sync_object_versions` ab und bewahrt ihn so lange auf wie Grabsteine (maximale Offline-Dauer plus Puffer, Vorschlag 60 Tage), und darüber hinaus, solange ein offener Konflikt darauf verweist. Für den Abgleich selbst sind diese Stände nicht nötig (die Basis kommt als Vorwerte mit), sie dienen der Konfliktanzeige („so sah es vorher aus“), dem Wiederherstellen eines gelöschten Objekts aus einem Konflikt heraus und der Fehlersuche. Alte Stände werden wie die aktuellen gesichert. Beim endgültigen Löschen eines Objekts werden sie mit entfernt; dieses Löschen wartet aber wie beim Grabstein (Abschnitt 5), bis kein offener Konflikt mehr auf das Objekt verweist.

### 9.3 Tests

Die Sync-Engine bekommt die gründlichsten Tests im Kern: Eigenschaftsbasierte Tests (zufällige Folgen von Änderungen auf mehreren simulierten Geräten, mit und ohne Verbindung) prüfen, dass am Ende alle Geräte denselben Stand haben und keine Änderung ohne Konflikteintrag verschwindet.

---

## 10. PWA-Grundgerüst

### 10.1 Aufbau

- App-Shell mit Navigation, Startseite (Widgets), Einstellungen, Geräteverwaltung, Konfliktliste, Benachrichtigungseinstellungen.
- Responsiv für Handy, Tablet und PC; Bedienung auch per Tastatur; Kontraste nach WCAG-Stufe AA als Ziel.
- Service Worker: cacht die Shell und die UI-Teile aktiver Plugins (versioniert); bei neuer Version Hinweis „Update verfügbar, neu laden“.
- Web-App-Manifest für die Installation auf dem Home-Bildschirm; auf iOS führt die Einrichtung durch „Zum Home-Bildschirm“, weil Web Push und dauerhafter Speicher dort davon abhängen (GK 3.7, 3.8).

### 10.2 Verschlüsselter Offline-Speicher

Umsetzung von GK 3.4.3:

- IndexedDB mit zwei Bereichen: **Schlüsselbereich** (Salt, Argon2id-Parameter, Datenschlüssel gewrappt mit Passwort-Schlüssel, Datenschlüssel gewrappt mit Geräteschlüssel, Zeitpunkt der letzten Online-Anmeldung) und **Datenbereich** (jeder Datensatz einzeln mit AES-256-GCM verschlüsselt, zufälliger 96-Bit-IV je Datensatz).
- Auch die Schlüssel der IndexedDB-Einträge (Objekt-IDs) werden nicht im Klartext gespeichert, sondern als HMAC der ID (mit einem per HKDF aus dem Datenschlüssel abgeleiteten Schlüssel), damit sich aus dem Speicher nicht einmal Anzahl und Art der Objekte je Plugin ablesen lassen. Die Typinformation steht nur im verschlüsselten Inhalt.
- **Entsperren:** Online durch die normale Anmeldung (danach holt die PWA den Geräteschlüssel), offline durch das Passwort. Nach 15 Minuten ohne Bedienung oder wenn die App in den Hintergrund geht und länger als 5 Minuten dort bleibt, wird der Schlüssel aus dem Speicher entfernt und die App gesperrt (Werte einstellbar).
- **Offline-Frist:** Ist die letzte Online-Anmeldung älter als 30 Tage (E10), entsperrt die PWA offline nicht mehr.
- **Abmelden:** löscht beide Bereiche; bei nicht synchronisierten Änderungen vorher Warnung mit der Möglichkeit, zuerst zu synchronisieren.
- **Warteschlange:** Änderungen landen zuerst verschlüsselt in einer Warteschlange und werden beim nächsten Kontakt gesendet. Die Oberfläche zeigt die Zahl ungesendeter Änderungen.
- **Ohne Offline-Zugriff:** Für Konten ohne die Freigabe „Offline erlaubt“ (z. B. Gäste) speichert die PWA keine Daten dauerhaft; dann gelten die normalen Online-Regeln.

---

## 11. Weitere Kerndienste

### 11.1 i18n

Umsetzung von GK 3.6: Übersetzungsdateien je Sprache und Plugin (`de.json`, `en.json`), Schlüssel auf Englisch (`shopping.list.empty`), ICU MessageFormat über `intl-messageformat`, Formatierung über `Intl`. Server-seitige Texte (Mails, Push) in der Sprache des Empfängers. CI prüft, dass alle Schlüssel in allen Sprachen vorhanden sind und keine Texte fest im Code stehen (Lint-Regel für UI-Komponenten).

### 11.2 Benachrichtigungen

- **Web Push:** VAPID-Schlüsselpaar wird bei der Einrichtung erzeugt (privater Teil als Schlüsseldatei, 11.3). Je Gerät eine Subscription. Inhalte knapp halten (z. B. „Neue Änderung in Einkaufsliste“), Details erst nach Öffnen der App, weil Push-Inhalte auf dem Sperrbildschirm sichtbar sein können. Subscriptions, die der Push-Dienst als abgelaufen oder ungültig meldet, werden gelöscht; die genauen Antwortcodes werden in der Umsetzung gegen das Web-Push-Protokoll (RFC 8030 [K12]) und das Verhalten der Push-Dienste geprüft.
- **Mail:** SMTP über das Relay mit TLS und Anmeldung, Absender mit eigener Domain. Mails enthalten keine privaten Inhalte, nur Hinweise und Links. Die Wahl des Relay-Anbieters erfolgt vor Phase 5; in Phase 1 wird mit einem lokalen Test-Mailserver (Container, fängt alle Mails ab) entwickelt.
- **Alarme** (Sicherheit, Betrieb) gehen an alle Admins über beide Wege und lassen sich nicht abschalten.

### 11.3 Schlüssel und Feldverschlüsselung

- **Schlüsseldateien** außerhalb der Datenbank, nur für den Dienstnutzer lesbar: Feldverschlüsselungs-Hauptschlüssel, VAPID-Privatschlüssel, HMAC-Schlüssel für Tokens. Sie werden getrennt von der Datenbank gesichert (GK 3.4.4); das genaue Verfahren gehört zu Phase 4.
- **Feldverschlüsselung** für sensible Felder (in Phase 1: TOTP-Geheimnisse, Geräteschlüssel, Plugin-Felder, die ein Plugin als sensibel markiert): AES-256-GCM, zufälliger IV, Schlüssel-ID im Chiffretext, damit der Schlüssel später gewechselt werden kann. Als zusätzliche authentisierte Daten (AAD) wird der vollständige Ort des Werts eingebunden: Datenbereich (z. B. `core.totp_credentials` oder Plugin-ID und Objekttyp), Objekt- bzw. Datensatz-ID und Feldpfad, eindeutig kodiert (z. B. als JSON-Array). Beim Entschlüsseln baut der Kern die AAD aus dem Ort auf, an dem er den Wert erwartet, nicht aus Angaben im Chiffretext. So schlägt die GCM-Prüfung fehl, wenn verschlüsselte Werte zwischen Datensätzen **oder zwischen Feldern desselben Datensatzes** vertauscht werden. Frühere Stände in `sync_object_versions` verwenden dieselbe AAD wie der aktuelle Stand, weil ein Stand beim Ablegen nicht neu verschlüsselt wird; das Vertauschen mit einem älteren Stand desselben Felds verhindert die AAD nicht, ist aber nur mit Schreibzugriff auf die Datenbank möglich.

### 11.4 Schutz der Schnittstelle

- Allgemeine Ratenbegrenzung je Sitzung und je IP-Präfix (z. B. 300 Anfragen pro Minute), strengere Grenzen für Anmeldung, Einladungen und Uploads.
- Jede Route hat ein JSON-Schema für Eingabe und Ausgabe; unbekannte Felder werden abgelehnt. Größenlimits für Anfragen (Standard 1 MiB, Plugin-Upload 50 MiB).
- Sicherheits-Header: Content Security Policy ohne `unsafe-inline` und ohne fremde Quellen, HSTS, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `frame-ancestors 'none'`, `Permissions-Policy` restriktiv.
- Fehlermeldungen nach außen ohne interne Details; Details nur im Log.

### 11.5 Jobs

In-Prozess-Planer, Aufträge in `core.jobs` gespeichert, damit sie einen Neustart überstehen. Ein Job läuft nie doppelt (Sperre per `SELECT … FOR UPDATE SKIP LOCKED`). Kern-Jobs in Phase 1: abgelaufene Sitzungen und Einladungen löschen, Grabsteine bereinigen, Push-Subscriptions aufräumen, Integritätsprüfung der Plugins (täglich).

### 11.6 Logging, Audit und Alarme

- **Logs:** strukturiert als JSON (Pino), ohne Passwörter, Tokens, Inhalte privater Objekte. IP-Adressen nur gekürzt (Präfix). Aufbewahrung 30 Tage.
- **Audit-Log:** nur anhängen, in der Datenbank; Einträge für Anmeldungen, Fehlversuche, Faktor- und Passwortänderungen, Geräte, Rechte, Admin-Aktionen, Plugin-Lebenszyklus. Die Datenbankrolle des Kerns bekommt auf `audit_log` nur `INSERT` und `SELECT` (für die Anzeige im Admin-Bereich, 11.7), kein `UPDATE`, `DELETE` oder `TRUNCATE`. Eigentümer der Tabelle ist die Migrationsrolle, nicht die Laufzeitrolle, weil ein Eigentümer seine Rechte sonst selbst wieder erweitern könnte. Das Löschen alter Einträge (nach 1 Jahr) läuft über eine Datenbankfunktion `core.audit_purge()` mit `SECURITY DEFINER`, die der Migrationsrolle gehört, einen festen `search_path` setzt und nur Einträge löscht, die älter als 1 Jahr sind (die Grenze steht in der Funktion, nicht in einem Parameter) [K16]. Die Laufzeitrolle darf nur diese Funktion ausführen, aber selbst nichts aus `audit_log` löschen. So kann auch ein kompromittierter Kernprozess keine jüngeren Einträge entfernen.
- **Alarm-Regeln** in Phase 1: Fehlversuche (6.3), neue Anmeldung eines Admins von einem neuen Gerät, Plugin-Integritätsfehler, wiederholte Fehler in Jobs.

### 11.7 Admin-Bereich

- Erreichbar nur, wenn die Anfrage aus dem Heimnetz kommt. Der Kern nimmt die Client-Adresse nur aus dem Header, den nginx setzt, und nur, wenn die Verbindung von nginx selbst (`127.0.0.1`) kommt; nginx überschreibt den Header immer, statt ihn anzuhängen (GK 3.4.2).
- **Was als Heimnetz gilt:** private IPv4-Bereiche (RFC 1918 [K10]) und eindeutige lokale IPv6-Adressen (ULA, `fc00::/7`, RFC 4193 [K11]), jeweils auf die tatsächlich genutzten Bereiche eingeschränkt (Konfiguration). Öffentliche IPv6-Adressen gelten nicht als Heimnetz, weil Anbieter das Präfix eines Anschlusses ändern können. Damit Geräte zu Hause den Server dann über eine private Adresse erreichen, muss der Servername im Heimnetz auf die private Adresse aufgelöst werden (lokaler DNS-Eintrag im Router). Ob der Router bei Portweiterleitungen die Absenderadresse externer Verbindungen erhält (und sie nicht durch eine private Adresse ersetzt), wird in Phase 5 am echten Router geprüft; sonst wäre die Heimnetz-Prüfung wirkungslos.
- Zusätzlich: Admin-Rolle und starke Anmeldung (6.4).
- Funktionen in Phase 1: Nutzer und Einladungen, Rollen, Gruppen, Geräte und Sitzungen aller Nutzer, Plugins, Signaturschlüssel, Audit-Log ansehen, Einstellungen, Systemstatus.

---

## 12. Schnittstellen für spätere Phasen

| Phase | Was der Kern schon vorsieht |
|---|---|
| 2 Einkaufslisten | Plugin-Schnittstelle, Sync mit Operationen, Freigaben, Push |
| 3 Kalender | URL-Bereich `/dav`, App-Passwörter (Tabelle und Verwaltung kommen in Phase 3), Konflikteinträge mit Löschabsicht (9.2), `ctx.http` für Google und iCloud |
| 4 Datensicherung | Hook „konsistenten Stand herstellen“ (Datenbank-Snapshot), Liste der zu sichernden Dateien und Schlüsseldateien |
| 5 Live-Betrieb | Gesundheits-Endpunkt (`/healthz`, nur lokal), Alarmkanäle, Konfiguration der Heimnetz-Bereiche |
| 7 Android-App | Sync-Protokoll ist unabhängig vom Browser; Anmeldung für native Apps (z. B. per Passkey) wird dort ergänzt |

---

## 13. Sicherheitsprüfung der Phase

Vor Abschluss von Phase 1:

- Durchgang durch die OWASP-Top-10-Kategorien und die Punkte dieses Dokuments mit Notizen im Pull Request.
- Automatische Tests für jede Sicherheitsregel: Drosselung, CSRF-Abwehr, Sitzungsablauf, Fernabmeldung, Rechtematrix, Plugin-Signatur (gültig, manipuliert, zurückgezogener Schlüssel, fremde Datei im Paket, Pfad mit `..`), Admin-Bereich von außen, Header.
- Abhängigkeiten ohne bekannte Schwachstellen (`npm audit`).

---

## 14. Akzeptanzkriterien

Phase 1 ist fertig, wenn:

1. Ein Admin per Kommandozeile angelegt werden kann und weitere Personen per Einladung beitreten.
2. Anmeldung per Passkey sowie per Passwort plus TOTP in Chromium und WebKit (Playwright) und auf je einem echten Android- und iOS-Gerät in der Testumgebung funktioniert.
3. Wiederherstellungscodes, Fernabmeldung und Geräteentfernung funktionieren.
4. Das Test-Plugin signiert, installiert, aktualisiert, deaktiviert und entfernt werden kann; ein manipuliertes Paket wird abgelehnt.
5. Zwei Geräte das Test-Plugin gleichzeitig und offline bearbeiten können und sich danach korrekt zusammenführen; Konflikte werden angezeigt und lassen sich auflösen.
6. Die PWA offline startet, nur nach Entsperren Daten zeigt, nach 30 Tagen ohne Online-Anmeldung nicht mehr offline entsperrt und beim Abmelden alles löscht.
7. Web Push auf Android (Chrome) und auf iOS (Home-Bildschirm-App) ankommt; Mails im Test-Mailserver landen.
8. Die Oberfläche vollständig auf Deutsch und Englisch verfügbar ist.
9. CI grün ist, Selbstreview und Copilot-Review keine offenen Befunde mehr haben.

---

## 15. Umsetzungsreihenfolge

Kleine Pull Requests, jeder mit Tests und Copilot-Review:

1. Monorepo, Tooling, CI, leere Fastify-App, PostgreSQL-Container, Migrationslogik.
2. Konfiguration, Schlüsseldateien, Feldverschlüsselung, Logging, Sicherheits-Header, CSRF-Abwehr, Ratenbegrenzung.
3. Konten, Einladungen, Passwort plus TOTP, Wiederherstellungscodes, Drosselung, Sitzungen, Geräte.
4. Passkeys.
5. Rollen, Gruppen, Freigaben, `can()` mit Rechtematrix-Tests.
6. Audit-Log, Alarme, Jobs.
7. PWA-Shell, i18n, Service Worker, HTTPS-Testumgebung.
8. Verschlüsselter Offline-Speicher, Entsperren, Geräteschlüssel.
9. Sync-Engine (Server), dann Sync-Client und Warteschlange.
10. Plugin-Host, Paketformat, Signierwerkzeug, Test-Plugin.
11. Web Push und Mail.
12. Admin-Bereich.
13. Gesamttest auf echten Geräten, Sicherheitsprüfung, Abschluss.

---

## 16. Offene Entscheidungen für Christian

| Nr. | Frage | Vorschlag |
|---|---|---|
| K-E1 | UI-Framework | Svelte (ohne SvelteKit) mit Vite |
| K-E2 | HTTPS in der Testumgebung | Eigene Test-Zertifizierungsstelle per Skript; Wurzelzertifikat auf den Testgeräten installieren |
| K-E3 | Rollen | Admin, Erwachsener, Kind, Gast wie in 7.1 |
| K-E4 | Sitzungslaufzeit | 30 Tage ohne Nutzung, höchstens 90 Tage |
| K-E5 | Automatisches Sperren der PWA | nach 15 Minuten ohne Bedienung bzw. 5 Minuten im Hintergrund, einstellbar |
| K-E6 | Mindestlänge Passwort mit Offline-Zugriff | 15 Zeichen (Passphrase), sonst 8 Zeichen |
| K-E7 | Private Objekte auch vor Admins in der Oberfläche verbergen | Ja (mit der in 7.2 genannten Grenze) |

---

## 17. Quellen

Alle abgerufen am 2026-11-08. Verweise „GK Qn“ beziehen sich auf die Quellen im Gesamtkonzept.

- [K1] Fastify, Validation and Serialization („Route validation relies on Ajv v8“). https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/
- [K2] Fastify, Logging („it uses pino as its logger“). https://fastify.dev/docs/latest/Reference/Logging/
- [K3] SimpleWebAuthn, Dokumentation (`@simplewebauthn/server`, `@simplewebauthn/browser`). https://simplewebauthn.dev/docs/
- [K4] RFC 6238, TOTP: Time-Based One-Time Password Algorithm (inkl. Testvektoren in Anhang B). https://www.rfc-editor.org/rfc/rfc6238
- [K5] Apple Support, Trust manually installed certificate profiles in iOS and iPadOS. https://support.apple.com/en-us/102390
- [K6] NIST SP 800-63B-4, Digital Identity Guidelines: Authentication and Authenticator Management (Passwortlänge, Verbot von Zusammensetzungsregeln, Begrenzung von Fehlversuchen). https://pages.nist.gov/800-63-4/sp800-63b.html
- [K7] OWASP Password Storage Cheat Sheet (Argon2id mindestens 19 MiB, t = 2, p = 1). https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- [K8] OWASP Cross-Site Request Forgery Prevention Cheat Sheet (SameSite als zusätzliche Maßnahme, `Sec-Fetch-Site` als Hauptsignal mit Pflicht-Rückfall auf Origin-Prüfung). https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- [K9] MDN, EventSource (Baseline „Widely available“; Grenze von 6 Verbindungen ohne HTTP/2). https://developer.mozilla.org/en-US/docs/Web/API/EventSource
- [K10] RFC 1918, Address Allocation for Private Internets. https://www.rfc-editor.org/rfc/rfc1918
- [K11] RFC 4193, Unique Local IPv6 Unicast Addresses. https://www.rfc-editor.org/rfc/rfc4193
- [K12] RFC 8030, Generic Event Delivery Using HTTP Push. https://www.rfc-editor.org/rfc/rfc8030
- [K13] PostgreSQL 18, CREATE SEQUENCE, Notes („Because nextval and setval calls are never rolled back …“) und Sequence Manipulation Functions (keine lückenlosen Folgen). https://www.postgresql.org/docs/current/sql-createsequence.html, https://www.postgresql.org/docs/current/functions-sequence.html
- [K14] PostgreSQL 18, Transaction Isolation, 13.2.1 Read Committed („the would-be updater will wait for the first updating transaction to commit or roll back … apply its operation to the updated version of the row“). https://www.postgresql.org/docs/current/transaction-iso.html
- [K15] Douglas Crockford, Base 32 (10 Ziffern und 22 Buchstaben, ohne I, L, O, U). https://www.crockford.com/base32.html
- [K16] PostgreSQL 18, CREATE FUNCTION, Abschnitt „Writing SECURITY DEFINER Functions Safely“ (festen `search_path` setzen, `EXECUTE` gezielt vergeben). https://www.postgresql.org/docs/current/sql-createfunction.html
- [K17] OWASP, Slow Down Online Guessing Attacks with Device Cookies (vertrauenswürdige Geräte einzeln drosseln, unbekannte Geräte gemeinsam je Konto). https://owasp.org/www-community/Slow_Down_Online_Guessing_Attacks_with_Device_Cookies
