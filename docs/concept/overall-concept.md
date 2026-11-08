# Gesamtkonzept Familienassistent

Stand: 2026-11-08 · Status: **Entwurf zur Freigabe** · Sprache dieses Dokuments: Deutsch (Programmcode durchgehend Englisch, siehe 3.6)

Dieses Dokument beschreibt das Gesamtbild über alle Phasen. Es legt Architektur, Grundsätze, Sicherheitsmodell und die Reihenfolge der Phasen fest. Details (Datenmodelle, APIs, Bildschirme) entstehen erst im Konzept der jeweiligen Phase. Aussagen zu externen Protokollen, Plattformen und Diensten sind mit Quellen belegt (Abschnitt 12, Verweise in eckigen Klammern wie [Q3]). Wo etwas nicht verifiziert werden konnte, steht das ausdrücklich dabei.

---

## 1. Ziel und Rahmen

**Ziel:** Ein selbst gehosteter Familienassistent auf einem eigenen Server zu Hause, der nach und nach alle Tätigkeiten und Probleme in Haus und Familie abdeckt. Die Daten sind sehr privat.

**Prioritäten (in dieser Reihenfolge, wenn sie kollidieren):**

1. Sicherheit und Schutz der privaten Daten.
2. Sauberer, leicht erweiterbarer Kern.
3. Funktionen als Plugins, die sich hinzufügen und entfernen lassen.

**Verbindliche Rahmenentscheidungen** (bereits mit Christian beschlossen):

| Thema | Entscheidung |
|---|---|
| Fremdsoftware | So wenig wie möglich. Alle Funktionen (auch Fotos, Backups, Smart Home) werden selbst entwickelt. Erlaubt sind etablierte Bibliotheken für Kryptografie, Datenbank, Web-Framework und CalDAV sowie Sync mit externen Diensten (Apple/Google, Online-Backup-Speicher, Mail-Relay). |
| Form | Responsive Webanwendung als PWA (Progressive Web App). |
| Nutzer | Familienmitglieder mit Rechtesystem. Endgeräte: Android (Christian) und iOS (seine Freundin). |
| Zugriff von außen | Öffentlich per HTTPS, kein VPN. |
| Offline | Alle Plugins offline lesen **und** bearbeiten. Offline-Daten im Speicher der PWA immer verschlüsselt, nur nach Anmeldung sichtbar, beim Abmelden gelöscht. Für Kalenderdaten, die über CalDAV oder die spätere Android-App im Kalender des Betriebssystems liegen, gilt eine Ausnahme (siehe 3.8, Entscheidung E12). |
| Vorgehen | Funktionen zuerst schlank bauen und testen, dann ausbauen. |
| Sprache | Kompletter Programmcode auf Englisch. Oberfläche mehrsprachig, zum Start Deutsch und Englisch. |
| Arbeitsweise | Je Phase: Konzept → Review bis problemfrei → Umsetzung → Review bis problemfrei → nächste Phase. Copilot Code Review auf jedem Pull Request, wiederholt bis Copilot nichts mehr findet. |

**Nicht-Ziele (vorerst):** Kein Betrieb für fremde Haushalte, keine Mandantenfähigkeit, keine nativen Apps außer der späteren Android-Sync-App, keine Prozess-Isolation von Plugins.

---

## 2. Phasenplan

Beschlossen sind die Phasen 0 bis 3. Die Reihenfolge ab Phase 4 ist ein **Vorschlag** (siehe Entscheidung E3 in Abschnitt 11).

| Nr. | Phase | Ergebnis | Abschlusskriterium |
|---|---|---|---|
| 0 | Gesamtkonzept | Dieses Dokument | Christian gibt frei |
| 1 | Kern | Server, Anmeldung, Nutzer und Rechte, Plugin-System, Sync- und Offline-Grundlage, i18n, Benachrichtigungen, Logging, Entwicklungs- und Testumgebung, CI | Alle Kernfunktionen mit automatischen Tests, Sicherheitsreview ohne offene Befunde |
| 2 | Einkaufslisten | Erstes Plugin, beweist Plugin-System und Offline-Sync an einem einfachen Fall | Mehrere Nutzer bearbeiten gleichzeitig und offline, Konflikte werden korrekt aufgelöst |
| 3 | Kalender | CalDAV-Server, Kalender in der PWA, Anbindung externer Kalender (Google, iCloud) | iOS-Kalender und DAVx⁵ synchronisieren fehlerfrei, PWA offline nutzbar |
| 4 | Datensicherung (Vorschlag) | Eigenes Backup mit Deduplizierung, Kompression, Verschlüsselung, Online-Kopie | Erfolgreicher Wiederherstellungstest auf leerem System |
| 5 | Live-Betrieb und Hardware (Vorschlag) | Server zu Hause, Domain, TLS, Härtung, Monitoring, USV, Notfallhandbuch | Sicherheitsprüfung von außen (Portscan, TLS-Test, Anmeldeangriffe) ohne Befunde, Notfallhandbuch von Christians Freundin nachvollzogen |
| 6 | Fotoalben | Foto- und Video-Verwaltung mit Alben und Freigaben | Upload vom Handy, Alben, Backup der Fotos geprüft |
| 7 | Android-Sync-App | Native App, die Kalender (und später weitere Daten) direkt mit dem Server synchronisiert und DAVx⁵ ablöst | Gleichwertiger Ersatz für DAVx⁵ im Alltag |
| 8 | Smart Home | Steuerung und Automationen für Geräte im Haus | Nur lokal steuerbar, sicherheitsgeprüft |

**Begründung der vorgeschlagenen Reihenfolge:** Echte, private Daten sollten erst dann dauerhaft auf dem Server liegen, wenn eine funktionierende, getestete Datensicherung existiert. Deshalb kommt die Datensicherung vor dem Live-Betrieb, und der Live-Betrieb vor den Fotoalben, weil Fotos den größten Teil der erwarteten rund 2 TB ausmachen dürften. Bis zum Live-Betrieb läuft alles in der lokalen Testumgebung.

Jede Phase durchläuft denselben Ablauf (Abschnitt 10). Eine Phase beginnt erst nach Christians Freigabe.

---

## 3. Architektur

### 3.1 Überblick

```
 Endgeräte                                   Heimserver
 ┌────────────────────┐   HTTPS   ┌──────────────────────────────────────────┐
 │ PWA (Browser)      │◄────────►│ Reverse Proxy / TLS (Let's Encrypt)       │
 │  - UI (i18n de/en) │           │        │                                 │
 │  - Service Worker  │           │        ▼                                 │
 │  - verschl. Offline│           │ Anwendungsserver (Kern)                  │
 │    -Speicher       │           │  ├─ Auth (Passkeys, Passwort+TOTP)       │
 ├────────────────────┤           │  ├─ Nutzer, Gruppen, Rechte              │
 │ iOS-Kalender,      │◄─CalDAV──►│  ├─ Plugin-Host (Signaturprüfung)        │
 │ DAVx⁵ (Android)    │           │  ├─ Sync-Engine + Konfliktlogik          │
 ├────────────────────┤           │  ├─ Benachrichtigungen (Web Push, Mail)  │
 │ Android-Sync-App   │◄─API─────►│  ├─ Jobs, Audit-Log, Monitoring          │
 │ (Phase 7)          │           │  └─ Plugins: Einkauf, Kalender, ...      │
 └────────────────────┘           │        │                                 │
                                  │        ▼                                 │
 Externe Dienste                  │ PostgreSQL (ein Schema je Plugin)        │
 ┌────────────────────┐           │ Dateispeicher (Fotos, Anhänge)           │
 │ Google / iCloud    │◄─────────►│ Backup-Dienst ──► Online-Speicher        │
 │ Push-Dienste       │◄──────────│                                          │
 │ Mail-Relay         │◄──────────│                                          │
 └────────────────────┘           └──────────────────────────────────────────┘
```

### 3.2 Technologie-Vorschlag (Entscheidung E1)

| Bereich | Vorschlag | Begründung |
|---|---|---|
| Sprache | TypeScript für Server und PWA | Eine Sprache für alles, gemeinsame Typen und Validierung zwischen Client und Server, Plugins bestehen aus Server- und UI-Teil in derselben Sprache. |
| Laufzeit | Node.js, jeweils aktuelle LTS-Version | Verbreitet, stabile Sicherheitsupdates, eingebaute Krypto (`node:crypto`). |
| Web-Framework | Fastify (oder vergleichbar schlank) | Schlank, Schema-Validierung eingebaut. Endgültige Wahl im Kern-Konzept. |
| Datenbank | PostgreSQL | Transaktionen, nebenläufige Nutzer, Schemata und Rollen für getrennte Plugin-Datenbereiche, ausgereifte Backup-Werkzeuge. |
| UI | Schlankes Komponenten-Framework (Kandidaten: Svelte, Preact, Lit) | Wahl im Kern-Konzept nach Bundle-Größe, Offline-Tauglichkeit und Testbarkeit. |
| iCalendar | ical.js | Parser und Wiederholungsregeln (RRULE) sind fehleranfällig; eine etablierte Bibliothek ist hier sicherer als Eigenbau. |
| Krypto im Browser | Web Crypto API für AES-GCM; Argon2id über eine etablierte WASM-Bibliothek | Argon2id wird für den Offline-Schlüssel gebraucht (3.4.3). |
| Android-App (Phase 7) | Kotlin | Plattformstandard. |
| Betriebssystem Server | Debian stable | Lange Sicherheitsunterstützung, `unattended-upgrades` für automatische Sicherheitsupdates. |

**Alternative, ehrlich bewertet:** Für Go gibt es mit `emersion/go-webdav` eine Bibliothek mit CalDAV-Unterstützung (Umfang wäre noch zu prüfen), und Go liefert eine einzelne Binärdatei. Dagegen spricht, dass Plugins zur Laufzeit nachgeladen werden sollen, was in Go nur eingeschränkt möglich ist, und dass Client und Server dann zwei Sprachen hätten. Bei TypeScript wird die CalDAV-Protokollschicht (WebDAV-Methoden, XML) selbst gebaut, was zum Grundsatz „selbst entwickeln“ passt, aber Aufwand und Testbedarf in Phase 3 erhöht.

### 3.3 Kern und Plugins

**Der Kern** enthält nur, was alle Plugins brauchen:

- Anmeldung, Sitzungen, Nutzer, Gruppen (z. B. „Familie“, „Erwachsene“), Rollen und Rechte.
- Plugin-Host: Installation, Signaturprüfung, Aktivierung, Deaktivierung, Entfernung, Migrationen.
- Datenzugriff je Plugin auf den eigenen Datenbereich.
- Sync-Engine und Konfliktlogik (3.5), gemeinsam für PWA, CalDAV und die spätere Android-App.
- Offline-Grundlage der PWA (Service Worker, verschlüsselter Speicher, Warteschlange für Änderungen).
- Benachrichtigungen (Web Push, Mail), zeitgesteuerte Jobs, Audit-Log, Einstellungen, i18n.
- Schnittstellen für Backup und Monitoring.

**Ein Plugin** ist ein Paket mit:

- `manifest.json`: ID, Version, benötigte Kernversion, Rechte (siehe unten), Übersetzungen.
- Server-Teil (Routen, Jobs, Sync-Typen), UI-Teil (Seiten, Widgets), Datenbank-Migrationen.
- Signatur (Ed25519) über den Hash aller Dateien.

**Rechte-Manifest:** Ein Plugin erklärt, was es braucht, z. B.:

- `storage:own` (eigener Datenbereich, immer),
- `read:calendar.events` (lesender Zugriff auf eine Schnittstelle eines anderen Plugins),
- `notify` (Benachrichtigungen senden),
- `jobs` (zeitgesteuerte Aufgaben),
- `net:www.googleapis.com` (ausgehende Verbindungen nur zu genannten Hosts).

Der Kern gibt einem Plugin nur die Schnittstellen, die im Manifest stehen und bei der Installation bestätigt wurden. Jedes Plugin bekommt ein eigenes PostgreSQL-Schema und eine eigene Datenbankrolle, die nur auf dieses Schema zugreifen darf. Plugins greifen nie direkt auf fremde Schemata zu, sondern nur über vom Kern vermittelte Schnittstellen.

**Wichtige Einschränkung (bewusst akzeptiert):** Weil Plugins vorerst im selben Prozess laufen (keine Prozess-Isolation), kann bösartiger Plugin-Code das Rechte-Manifest technisch umgehen, etwa über direkten Dateizugriff. Das Manifest schützt vor **Fehlern** und begrenzt die Folgen von Bugs. Gegen **bösartigen** Code schützt die Signaturpflicht: Es laufen nur Plugins, die Christian mit seinem privaten Schlüssel signiert hat. Eine spätere echte Isolation bleibt durch die klare Schnittstelle möglich, ohne Plugins neu zu schreiben: je Plugin ein eigener Prozess mit ausdrücklich eingeschränkten Betriebssystemrechten (z. B. eigener Systemnutzer, kein Zugriff auf fremde Dateien, Netz nur zu freigegebenen Hosts). Worker-Threads in Node.js wären dagegen keine Sicherheitsgrenze, weil sie im selben Prozess mit denselben Rechten laufen und Datei- und Netzzugriff behalten; sie trennen nur die Ausführung (Nebenläufigkeit), nicht die Rechte.

**Signaturen:** Der private Signaturschlüssel liegt nur auf Christians Rechner (plus ausgedruckte oder Offline-Sicherung), nie auf dem Server und nie in GitHub. Der Server kennt nur die öffentlichen Schlüssel. Er prüft die Signatur bei der Installation und bei jedem Start. Das gleiche Verfahren gilt für Updates des Kerns. Das Signieren ist damit ein bewusster manueller Schritt auf Christians Rechner. Damit dort nichts Unbekanntes signiert wird, baut das Signierwerkzeug das Paket lokal aus dem geprüften Git-Stand oder vergleicht den Hash mit dem CI-Build. Ein Schlüsselwechsel und das Zurückziehen eines Schlüssels werden im Kern-Konzept beschrieben.

**Entfernen eines Plugins:** Beim Entfernen wählt der Admin, ob die Daten exportiert, aufbewahrt oder gelöscht werden. Der Kern und andere Plugins laufen ohne das Plugin weiter. Hängt ein anderes Plugin von dessen Schnittstelle ab, warnt der Kern vorher.

Plugins installieren, aktualisieren und entfernen darf nur ein Admin, und nur aus dem Heimnetz. Den UI-Teil eines Plugins lädt die PWA ausschließlich vom eigenen Server; die Content Security Policy verbietet fremde Skriptquellen.

Einkaufslisten und Kalender werden selbst als Plugins gebaut. So wird das Plugin-System von Anfang an an echten Fällen erprobt.

### 3.4 Sicherheit

#### 3.4.1 Bedrohungsmodell

| Angreifer / Risiko | Maßnahmen |
|---|---|
| Angreifer aus dem Internet | Nur Port 443 offen (Port 80 höchstens für die Zertifikatsprüfung, siehe 8), Passkeys oder Passwort plus TOTP, gedrosselte Anmeldeversuche, Admin-Bereich nur aus dem Heimnetz, automatische Sicherheitsupdates, minimale Angriffsfläche, Logging mit Alarm. |
| Gestohlenes oder verlorenes Handy | Offline-Daten der PWA verschlüsselt, Sitzungen aus der Ferne beendbar, Geräteschlüssel auf dem Server löschbar, gerätebezogene App-Passwörter für CalDAV einzeln widerrufbar. Kalenderkopien im Betriebssystem schützt nur die Gerätesperre (3.8, E12). |
| Fehler in eigenem Code | Automatische Tests, Reviews (eigenes und Copilot), Rechte-Manifest, getrennte Datenbankrollen. |
| Kompromittierte Bibliothek (Lieferkette) | Wenige Abhängigkeiten, Lockfile mit festen Versionen, automatische Hinweise auf bekannte Schwachstellen, Updates nur per Pull Request mit Review. |
| Untergeschobenes Plugin oder Update | Nur signierte Pakete, Schlüssel nicht auf dem Server. |
| Familienmitglied überschreitet Rechte | Rechteprüfung serverseitig bei jedem Zugriff, nie nur in der Oberfläche. |
| Online-Backup-Anbieter | Daten werden vor dem Upload verschlüsselt, der Anbieter sieht nur Chiffretext. |
| Push-Dienste (Google, Apple, Mozilla) | Web-Push-Nachrichten sind Ende-zu-Ende verschlüsselt [Q1]; der Push-Dienst sieht nur Metadaten. Inhalte trotzdem knapp halten. |
| Diebstahl des Servers / der Platten | Festplattenverschlüsselung (Details und Entsperren nach Stromausfall in Phase 5, Entscheidung E8), zusätzlich Feldverschlüsselung sensibler Daten. |

#### 3.4.2 Anmeldung und Sitzungen

- **Passkeys (WebAuthn)** als bevorzugtes Verfahren. WebAuthn Level 3 ist seit dem 25. August 2026 W3C-Empfehlung [Q2].
- **Passwort plus TOTP** als zweiter Weg (TOTP nach RFC 6238 [Q3]). Passwörter werden serverseitig mit Argon2id gehasht [Q4].
- **Wiederherstellungscodes** einmalig nutzbar, nur gehasht gespeichert.
- **Drosselung statt harter Sperre:** Nach Fehlversuchen steigt die Wartezeit je Konto und je IP-Adresse an. Eine harte Kontosperre würde Angreifern erlauben, Familienmitglieder absichtlich auszusperren. Bei auffälligen Mustern gibt es einen Alarm an den Admin. (Die beschlossene „Sperre nach Fehlversuchen“ wird so umgesetzt; falls Christian eine echte Sperre möchte, siehe E9.)
- **Sitzungen:** Liste aller Sitzungen und Geräte je Nutzer, einzeln oder alle aus der Ferne beendbar. Sitzungs-Cookies `HttpOnly`, `Secure`, `SameSite` (`Strict` oder `Lax`, wird im Kern-Konzept mit Links aus Mails getestet), zusätzlich CSRF-Schutz.
- **App-Passwörter für CalDAV:** iOS-Kalender und DAVx⁵ können keine Passkeys. Für sie erzeugt der Nutzer pro Gerät ein zufälliges App-Passwort, das nur für CalDAV gilt und einzeln widerrufen werden kann. Apple nutzt für iCloud dasselbe Prinzip [Q5].
- **Admin-Bereich nur aus dem Heimnetz:** Prüfung der Quelladresse auf dem Server. Im Kern-Konzept ist festzulegen, wie das mit Reverse Proxy, IPv6 und weitergeleiteten Headern fälschungssicher gelöst wird (der Header `X-Forwarded-For` darf nur vom eigenen Proxy übernommen werden).

#### 3.4.3 Offline-Verschlüsselung (beschlossenes Verfahren, ergänzt um einen Geräteschlüssel, Entscheidung E14)

1. Beim ersten Login auf einem Gerät erzeugt die PWA einen zufälligen **Datenschlüssel** (AES-256-GCM).
2. Aus dem Passwort und einem zufälligen Salt wird per **Argon2id** ein Schlüssel abgeleitet, der den Datenschlüssel verschlüsselt („wrappt“). Gespeichert werden nur Salt, Parameter und der gewrappte Datenschlüssel, **kein Passwort-Hash**.
   Zusätzlich wird der Datenschlüssel ein zweites Mal gewrappt, mit einem zufälligen **Geräteschlüssel**, den nur der Server speichert (je Gerät einer). Auf dem Gerät liegt nur diese zweite gewrappte Kopie, nicht der Geräteschlüssel selbst. Der Server gibt den Geräteschlüssel nur nach einer erfolgreichen Online-Anmeldung dieses Geräts heraus (Passwort plus TOTP oder Passkey). Das verschlechtert den Schutz nicht, weil der Server alle synchronisierten Daten ohnehin besitzt; es ermöglicht aber die Schritte 6 und 7.
3. Offline-Anmeldung: Das eingegebene Passwort wird erneut abgeleitet. Nur wenn das Entschlüsseln des Datenschlüssels gelingt (der GCM-Authentisierungs-Tag passt), ist das Passwort richtig.
4. Alle Offline-Daten (IndexedDB) sind mit dem Datenschlüssel verschlüsselt. Der entschlüsselte Schlüssel liegt nur im Arbeitsspeicher, als nicht exportierbarer Web-Crypto-Schlüssel.
5. Beim Abmelden werden alle Offline-Daten und Schlüssel gelöscht. Gibt es noch nicht synchronisierte Änderungen, warnt die PWA vorher.
6. **Passwort geändert oder zurückgesetzt:** Ein anderes Gerät kennt das neue Passwort nicht und kann den Datenschlüssel damit nicht entsperren. Bei der nächsten Online-Anmeldung auf diesem Gerät (mit dem neuen Passwort oder einem Passkey) holt die PWA den Geräteschlüssel vom Server, entsperrt damit den Datenschlüssel und wrappt ihn neu mit dem neuen Passwort. Das alte Passwort ist dafür nicht nötig, und der verschlüsselte Offline-Speicher samt nicht synchronisierter Änderungen bleibt erhalten. Bis zu dieser Online-Anmeldung entsperrt auf dem Gerät weiterhin nur das alte Passwort; die Oberfläche weist darauf hin.
7. **Gerät gesperrt oder entfernt:** Der Server löscht den Geräteschlüssel. Ist das Gerät offline, bleiben die Daten dort nur noch mit dem alten Passwort entsperrbar, bis das Gerät wieder Kontakt hat und seine Daten löscht, oder bis die maximale Offline-Dauer abläuft (siehe unten).

**Fernabmeldung und Offline-Geräte:** Eine aus der Ferne beendete Sitzung kann ein Gerät, das gerade offline ist, nicht sofort löschen. Das Gerät löscht seine Daten, sobald es wieder Kontakt zum Server hat. Zusätzlich gilt eine maximale Offline-Dauer (Vorschlag: 30 Tage, Entscheidung E10), nach der die PWA erst nach einer Online-Anmeldung wieder entsperrt. Grenze: Diese Frist setzt die PWA selbst durch. Wer die gespeicherten Daten aus dem Browser kopiert und das alte Passwort kennt oder errät, kann sie auch danach entschlüsseln; die Frist schützt also vor normaler Nutzung, nicht vor einem gezielten Angriff mit kopierten Daten.

**Passwort nötig für Offline-Zugriff:** Weil der Schlüssel aus dem Passwort abgeleitet wird, braucht jedes Konto mit Offline-Zugriff ein Passwort, auch wenn online meist der Passkey genutzt wird.

**Parameter:** RFC 9106 empfiehlt als zweite Option Argon2id mit t=3, p=4 und 64 MiB Speicher, wenn 2 GiB nicht praktikabel sind [Q4]. 2 GiB sind im Handy-Browser nicht realistisch; geplant ist daher mindestens die zweite Empfehlung. Die endgültigen Werte werden im Kern-Konzept auf echten Geräten gemessen (Ziel: Ableitung unter etwa einer Sekunde).

**Grenze des Verfahrens:** Wer das Gerät samt Browserspeicher besitzt, kann offline beliebig viele Passwörter durchprobieren. Argon2id macht jeden Versuch teuer, die Sicherheit hängt aber von der Passwortstärke ab. Die PWA verlangt daher für Konten mit Offline-Zugriff ein ausreichend langes Passwort. Als spätere Option kann die WebAuthn-Erweiterung `prf` (Teil von Level 3 [Q2]) den Schlüssel aus einem Passkey ableiten; ob iOS und Android sie zuverlässig unterstützen, ist noch nicht geprüft.

#### 3.4.4 Weitere Pflichtmaßnahmen

- **TLS:** Let's-Encrypt-Zertifikat mit automatischer Verlängerung. Die Standardlaufzeit beträgt derzeit 90 Tage; Let's Encrypt senkt die Höchstlaufzeit bis Februar 2028 auf 45 Tage [Q6]. Die Verlängerung muss daher vollautomatisch laufen und überwacht werden.
- **Zusätzliche Verschlüsselung sensibler Daten** in der Datenbank (Feldverschlüsselung). Die Schlüssel liegen getrennt von der Datenbank und werden nicht zusammen mit dem Datenbank-Backup gespeichert, sondern gesondert gesichert. Welche Felder als sensibel gelten, legt jedes Phasenkonzept fest.
- **Sicherheits-Header:** strikte Content Security Policy, HSTS, keine Inline-Skripte.
- **Logging und Alarm:** Anmeldungen, Fehlversuche, Rechteänderungen, Admin-Aktionen und Plugin-Installationen ins Audit-Log; Alarme per Push und Mail.

### 3.5 Synchronisation und Konflikte

Grundsatz: **Der Server ist die einzige verbindliche Quelle.** Alle Wege (PWA, CalDAV, später Android-App) laufen durch dieselbe Sync-Engine im Kern.

- Jedes Objekt hat eine ID, eine **Revision** (fortlaufender Zähler des Servers) und einen Zeitstempel.
- Clients holen Änderungen „seit Revision X“. Bei CalDAV entspricht das dem `sync-token` aus RFC 6578 [Q7], die Objekt-Revision dem `ETag`.
- Clients merken sich den Stand, auf dem eine Änderung beruht (Basis), und senden die Änderung mit dieser Revision und einer eindeutigen Änderungs-ID, damit ein wiederholtes Senden nichts doppelt ausführt.
- **Konfliktlösung im Kern:** Hat sich das Objekt seitdem geändert, führt der Server einen Drei-Wege-Abgleich auf Feldebene durch (Basis, Server, Client). Wurden verschiedene Felder geändert, werden beide Änderungen übernommen. Zwei Schutzregeln verhindern dabei ungültige Ergebnisse: Plugins fassen voneinander abhängige Felder zu einer **Konflikteinheit** zusammen (z. B. Beginn, Ende, Dauer, Zeitzone und Wiederholungsregel eines Termins), die nur als Ganzes übernommen wird. Außerdem prüft das Plugin jedes zusammengeführte Ergebnis auf seine Regeln (z. B. Ende nicht vor Beginn). Schlägt die Prüfung fehl, gilt das wie eine Änderung desselben Felds: Der Serverstand bleibt, und die Client-Version wird Konfliktkopie. Beispiel: Ein Termin 9:00–10:00, ein Gerät verschiebt den Beginn auf 9:30, ein anderes das Ende auf 9:15; ohne diese Regeln entstünde 9:30–9:15. Wurde dasselbe Feld geändert, bleibt der Serverstand gültig, und die Client-Version wird als **Konfliktkopie** gespeichert, die der Nutzer sieht und auflösen kann. Es geht also nie stillschweigend etwas verloren.
- Plugins können für eigene Datentypen feinere Regeln festlegen, etwa Einkaufslisten als einzelne Operationen („Artikel hinzufügen“, „abhaken“), die sich fast immer konfliktfrei zusammenführen lassen.
- **CalDAV-Besonderheit:** CalDAV-Clients senden Änderungen mit `If-Match` und dem bekannten `ETag`. Passt der `ETag` nicht mehr, antwortet der Server nach HTTP-Standard mit `412 Precondition Failed` [Q23]. Das schützt den Serverstand davor, überschrieben zu werden. Was der Client danach mit seiner abgelehnten Änderung macht, legt der Standard nicht fest (z. B. erneut versuchen, verwerfen oder dem Nutzer anzeigen). Damit trotzdem nichts verloren geht, **speichert der Server die abgelehnte Änderung selbst als Konflikteintrag**, bevor er mit 412 antwortet. Der Eintrag hält fest, welche Art von Änderung abgelehnt wurde: bei `PUT` den gesendeten Inhalt, bei `DELETE` (das keinen Inhalt mitschickt) die Objekt-ID, den `ETag`, auf dem der Client beruhte, und die Absicht „löschen“. Der Konflikteintrag ist kein CalDAV-Objekt, damit Clients ihn nicht als zweiten Termin anzeigen; der Nutzer sieht ihn in der PWA und kann die Änderung bzw. das Löschen übernehmen oder verwerfen. Wie sich iOS-Kalender und DAVx⁵ bei gleichzeitigen Offline-Änderungen tatsächlich verhalten (etwa ob sie die Änderung erneut senden und so doppelte Konflikteinträge entstehen), getrennt für die Fälle „Ändern gegen Ändern“ und „Ändern gegen Löschen“ in beide Richtungen,, wird in Phase 3 gezielt getestet und im Kalender-Konzept dokumentiert.
- **Keine Hintergrund-Synchronisation unter iOS:** Die Background Sync API wird von Safari nicht unterstützt [Q9]. Die PWA synchronisiert daher beim Öffnen, beim Wiederherstellen der Verbindung und periodisch, solange sie offen ist. Nicht synchronisierte Änderungen zeigt die Oberfläche deutlich an.

### 3.6 Sprache im Code, Mehrsprachigkeit der Oberfläche

- **Code komplett auf Englisch:** Bezeichner, Kommentare, Log-Meldungen, API-Namen, Datenbanknamen, Dateinamen. Vorschlag: auch Commit-Messages, Pull-Request-Titel und technische Code-Dokumentation auf Englisch (Entscheidung E5). Konzeptdokumente wie dieses bleiben auf Deutsch.
- **i18n im Kern von Anfang an:** Keine festen Texte in der Oberfläche. Alle Texte kommen aus Übersetzungsdateien je Sprache (ICU-MessageFormat für Plural und Platzhalter). Plugins bringen ihre eigenen Übersetzungen mit. Datum, Uhrzeit, Zahlen und Währungen formatiert die eingebaute `Intl`-API des Browsers.
- **Sprachen zum Start:** Deutsch und Englisch. Weitere Sprachen lassen sich ohne Codeänderung ergänzen.
- Jeder Nutzer wählt seine Sprache; Standard ist die Browsersprache. Auch Push-Nachrichten und Mails werden in der Sprache des Empfängers erzeugt.
- Ein automatischer Test prüft, dass jeder Text in allen Sprachen vorhanden ist.
- Zeitpunkte (z. B. „erstellt am“) werden in UTC gespeichert.
- Termine folgen den Zeitarten von iCalendar [Q13] und werden verlustfrei so gespeichert, wie sie angelegt wurden:
  - **Termine mit Uhrzeit an einem Ort** in lokaler Zeit mit Zeitzonen-ID (Standard `Europe/Berlin`), damit ein wöchentlicher Termin um 9 Uhr auch nach der Zeitumstellung um 9 Uhr bleibt.
  - **Ganztägige Termine** als reines Datum ohne Uhrzeit und ohne Zeitzonen-ID. RFC 5545 verbietet den Parameter `TZID` bei reinen Datumswerten (Abschnitt 3.2.19); als Mitternacht in einer Zeitzone gespeichert würde ein Geburtstag in einer anderen Zeitzone auf den falschen Tag rutschen.
  - **UTC-Zeiten** und **„schwebende“ Zeiten** (ohne Zeitzone, gelten überall zur selben lokalen Uhrzeit) bleiben als solche erhalten und werden nicht umgerechnet.

### 3.7 Benachrichtigungen

- **Web Push** als Hauptweg (Push API mit Service Worker, Verschlüsselung nach RFC 8291 [Q1], Server-Identifikation mit VAPID nach RFC 8292 [Q10]).
- **iOS:** Web Push gibt es dort ab iOS/iPadOS 16.4 und nur für Web-Apps, die zum Home-Bildschirm hinzugefügt wurden. Die Erlaubnis muss durch eine direkte Nutzeraktion angefragt werden [Q11]. Die Einrichtung erklärt das Schritt für Schritt. Hinweis: Die Quelle beschreibt die Beta von 16.4; dass Web Push in der aktuellen iOS-Version wie beschrieben funktioniert, wird in Phase 1 auf dem iPhone von Christians Freundin geprüft.
- **Mail** über einen externen Anbieter oder Relay mit eigener Domain, mit SPF, DKIM und DMARC, damit Mails nicht im Spam landen.
- Nutzer stellen je Plugin und Ereignis ein, ob und wie sie benachrichtigt werden.

### 3.8 Offline-Verhalten der PWA

- Service Worker liefert die Anwendung offline aus; Daten liegen verschlüsselt in IndexedDB (3.4.3).
- **Speicherbereinigung bei iOS:** Safari löscht skriptbeschreibbaren Speicher (u. a. IndexedDB und Service-Worker-Registrierungen) nach sieben Tagen Safari-Nutzung ohne Interaktion mit der Seite. Zum Home-Bildschirm hinzugefügte Web-Apps haben laut WebKit einen eigenen Zähler, und WebKit erwartet nicht, dass deren Daten gelöscht werden [Q12]. Daher gilt: Die PWA wird auf iOS immer über den Home-Bildschirm genutzt. Weil der Browser Speicher trotzdem bei Platzmangel freigeben kann, darf der Offline-Speicher nie die einzige Kopie sein; die PWA synchronisiert so früh wie möglich und zeigt ungesicherte Änderungen an.
- **Ausnahme Kalender im Betriebssystem (Entscheidung E12):** Über CalDAV (iOS-Kalender, DAVx⁵) und später die Android-Sync-App landen Termine im Kalenderspeicher des Betriebssystems. Diese Kopien verschlüsselt nicht die PWA, sie sind für Kalender-Apps auf dem Gerät lesbar, und das Abmelden in der PWA oder das Widerrufen des App-Passworts löscht sie nicht. Die Zusage „verschlüsselt, nur nach Anmeldung sichtbar, beim Abmelden gelöscht“ gilt daher nur für den Speicher der PWA. Vorschlag für diese Ausnahme:
  - Kalender-Sync aufs Betriebssystem nur auf Geräten mit Bildschirmsperre und eingeschalteter Geräteverschlüsselung.
  - Sensible Kalender können vom CalDAV-Sync ausgeschlossen werden und sind dann nur in der PWA sichtbar.
  - Beim Entfernen eines Geräts muss das Konto auf dem Gerät gelöscht werden (in DAVx⁵ bzw. in den iOS-Einstellungen). Ob dabei die lokal gespeicherten Termine zuverlässig mit gelöscht werden, ist nicht verifiziert und wird in Phase 3 auf beiden Systemen geprüft. Die eigene Android-App (Phase 7) löscht ihre Termine beim Entfernen des Kontos und, sobald der Server das Gerät gesperrt hat, beim nächsten Kontakt.

---

## 4. Kern – Phase 1 (Umriss)

Umfang:

- Projektstruktur (Monorepo), Build, Linting, Formatierung, automatische Tests (Unit, Integration, Ende-zu-Ende mit Playwright).
- Lokale Testumgebung, die den späteren Server nachbildet (Container mit PostgreSQL), inklusive Testdaten. Sie ist im Heimnetz per HTTPS erreichbar, damit echte Handys testen können: Service Worker, Web Push und Passkeys funktionieren nur in einem sicheren Kontext (HTTPS), und iOS verlangt für CalDAV eine verschlüsselte Verbindung [Q14]. Wie das Testzertifikat entsteht (eigene Test-Domain oder eigene Zertifizierungsstelle), legt das Kern-Konzept fest.
- Anmeldung (Passkeys, Passwort plus TOTP, Wiederherstellungscodes), Sitzungsverwaltung, Drosselung.
- Nutzer, Gruppen, Rollen (Vorschlag: Admin, Erwachsener, Kind, Gast) und Rechte je Plugin und Objekt (z. B. eine Liste nur für bestimmte Personen).
- Plugin-Host mit Manifest, Signaturprüfung, Migrationen, Aktivieren und Entfernen; Signierwerkzeug für Christians Rechner.
- Sync-Engine mit Konfliktlogik und Konfliktkopien.
- PWA-Gerüst mit Offline-Verschlüsselung, Warteschlange, i18n (de, en).
- Benachrichtigungen (Web Push, Mail), Jobs, Audit-Log, Admin-Bereich.

Bewusst nicht im Kern: fachliche Funktionen (die kommen als Plugins).

Geplante Repository-Struktur (Namen englisch):

```
/core            Server-Kern
/web             PWA-Grundgerüst
/plugins/shopping
/plugins/calendar
/packages/shared Gemeinsame Typen, Validierung, i18n-Werkzeuge
/tools/signer    Signierwerkzeug (läuft nur auf Christians Rechner)
/android         Android-Sync-App (Phase 7)
/docs            Konzepte, Entscheidungen, Notfallhandbuch
```

## 5. Einkaufslisten – Phase 2 (Umriss)

- Mehrere Listen, je Liste festlegbar, wer sie sieht und bearbeitet.
- Artikel mit Menge, Einheit, optional Kategorie (z. B. nach Gang im Supermarkt sortierbar), abhaken, rückgängig machen.
- Änderungen als Operationen, damit gleichzeitiges Bearbeiten (auch offline) konfliktfrei zusammengeführt wird.
- Live-Aktualisierung auf anderen Geräten, solange die App offen ist; optional Benachrichtigung bei Änderungen.
- Schlanker Start; Ausbau (z. B. Vorschläge aus häufigen Artikeln) erst nach Test im Alltag.

## 6. Kalender – Phase 3 (Umriss)

- **Eigener CalDAV-Server** (RFC 4791 [Q8]) als verbindliche Quelle, mit `sync-collection` (RFC 6578 [Q7]) für effiziente Synchronisation. Daten im iCalendar-Format (RFC 5545 [Q13]).
- **iOS:** Der eingebaute Kalender kann CalDAV-Konten direkt einbinden, ohne Zusatz-App (laut [Q14]: Einstellungen → Apps → Kalender → Kalenderaccounts → Account hinzufügen → Andere → CalDAV-Account; die genaue Menübezeichnung hängt von der iOS-Version ab). iOS verlangt dafür eine verschlüsselte Verbindung [Q14].
- **Android:** Übergangsweise DAVx⁵. DAVx⁵ synchronisiert CalDAV/CardDAV-Sammlungen mit Android und legt die Termine im Android-Kalenderspeicher ab, sodass jede Kalender-App sie anzeigt; DAVx⁵ ist Open Source unter GPLv3 [Q15]. (Ob Android ab Werk CalDAV kann, ist herstellerabhängig und wurde nicht verifiziert; für die Planung wird davon ausgegangen, dass nicht.)
- **Kalender in der PWA:** Tages-, Wochen-, Monatsansicht, offline bearbeitbar, Wiederholungen, Erinnerungen per Web Push.
- **Externe Kalender:**
  - **Google:** Google bietet eine CalDAV-Schnittstelle an, die zwingend OAuth 2.0 verlangt; Basic Auth wird mit 401 abgelehnt [Q16]. Alternativ die Google Calendar API. **Wichtig:** Solange die OAuth-App im Status „Testing“ mit externem Nutzertyp ist, laufen Refresh-Tokens nach 7 Tagen ab [Q17]. Für den Dauerbetrieb muss die App auf „In production“ gestellt werden. Ob und welche Prüfung durch Google dafür bei Kalender-Berechtigungen nötig ist, wird im Kalender-Konzept geklärt (nicht verifiziert).
  - **iCloud:** Zugriff per CalDAV mit app-spezifischem Passwort; dafür muss das Apple-Konto Zwei-Faktor-Authentifizierung haben [Q5].
  - **Richtung:** Vorschlag, zunächst nur Einlesen (externe Kalender werden im Familienkalender angezeigt) und Zwei-Wege-Sync erst später (Entscheidung E4).
- Kompatibilitätstests mit echten Clients (iOS-Kalender, DAVx⁵) sind Pflicht, weil Clients CalDAV unterschiedlich auslegen.

## 7. Datensicherung – Phase 4 (Umriss)

- **Eigenes Backup-Programm:** Inhaltsbasierte Aufteilung in Blöcke (Deduplizierung), Kompression mit zstd, danach Verschlüsselung mit einem authentisierten Verfahren (AES-256-GCM oder XChaCha20-Poly1305 aus einer etablierten Bibliothek). Reihenfolge „erst komprimieren, dann verschlüsseln“ ist beschlossen, weil verschlüsselte Daten sich nicht mehr komprimieren lassen.
- Mögliches Restrisiko, im Phasenkonzept zu bewerten: Bei deduplizierenden, komprimierten Backups könnte ein Speicheranbieter aus Anzahl und Größe der Blöcke Rückschlüsse auf Inhalte ziehen. Gegenmaßnahmen (z. B. Auffüllen auf feste Größen) werden dort untersucht.
- **Datenbank** konsistent sichern (Snapshot bzw. `pg_dump`), nicht nur Dateien kopieren.
- **3-2-1:** drei Kopien, zwei Medien, eine außer Haus (Online-Speicher, verschlüsselt von Anfang an). Wichtig: RAID ist kein Backup. Als zweites lokales Medium wird zusätzlich eine externe Platte gebraucht (Hardware-Liste in Phase 5).
- **Dauer der ersten Online-Sicherung:** 2 TB bei 50 Mbit/s dauern rechnerisch rund 3,7 Tage Dauer-Upload (2 × 10¹² Byte × 8 / 50 × 10⁶ bit/s ≈ 320 000 s), in der Praxis länger. Danach laufen nur noch Änderungen. Der Upload wird gedrosselt, damit der Anschluss nutzbar bleibt.
- **Schlüsselmodell (Entscheidung E13):** Ein Backup, das nachts unbeaufsichtigt läuft, braucht auf dem Server einen Schlüssel zum Verschlüsseln. Damit trotzdem der Entschlüsselungsschlüssel nur bei Christian liegt, wird ein Verfahren mit öffentlichem Schlüssel vorgeschlagen:
  - Für jeden Backup-Lauf erzeugt der Server zufällige Datenschlüssel und verschlüsselt sie mit dem **öffentlichen Wiederherstellungsschlüssel**. Der zugehörige **private Wiederherstellungsschlüssel** liegt nur bei Christian und ausgedruckt; der Server kann seine eigenen Backups nicht entschlüsseln.
  - Für die Deduplizierung braucht der Server einen eigenen **Dedup-Schlüssel**, mit dem er Block-Kennungen berechnet (z. B. per HMAC). Er erlaubt kein Entschlüsseln. Wer ihn erbeutet, könnte aber prüfen, ob ein bestimmter, ihm bekannter Inhalt im Backup vorkommt; das ist im Phasenkonzept zu bewerten.
  - Der Vorteil gegenüber einem einzigen symmetrischen Schlüssel auf dem Server: Wer später den Server übernimmt, kann ältere Backups beim Speicheranbieter nicht entschlüsseln, also auch keine dort noch vorhandenen, auf dem Server längst gelöschten Daten.
  - Die konkreten Verfahren (z. B. X25519 mit HKDF) aus einer etablierten Krypto-Bibliothek werden im Phasenkonzept festgelegt.
- **Wiederherstellungstests:** Automatisch prüft der Server regelmäßig, ohne zu entschlüsseln, ob alle Blöcke beim Speicheranbieter vorhanden und unverändert sind (Prüfsummen über den Chiffretext) und ob die Verzeichnisse vollständig sind. Ein echter Entschlüsselungstest braucht den privaten Schlüssel und läuft deshalb manuell: Christian stellt mindestens vierteljährlich Stichproben und mindestens einmal im Jahr das ganze System auf einem getrennten Rechner wieder her (Vorschlag).
- **Schlüssel** (privater Wiederherstellungsschlüssel) nur bei Christian und ausgedruckt an einem sicheren Ort. Ohne Schlüssel ist das Backup wertlos, deshalb gehört das Wiederherstellen ins Notfallhandbuch. **Zielkonflikt:** Wenn Christian ausfällt, kann seine Freundin ohne Zugang zum ausgedruckten Schlüssel nichts wiederherstellen. Vorschlag: Der Ausdruck liegt versiegelt an einem Ort, den das Notfallhandbuch nennt (Entscheidung E11).
- Wahl des Online-Speichers (z. B. S3-kompatibel) im Phasenkonzept.

## 8. Live-Betrieb und Hardware – Phase 5 (Umriss)

- **Hardware (festgehalten):** RAID 6 mit 4 × 4 TB, Gehäuse mit mindestens 4 Einschüben, USV; zusätzlich vorgeschlagen eine externe Platte für die lokale Backup-Kopie (7). RAID 6 mit vier Platten ergibt rund 8 TB (etwa 7,3 TiB) nutzbar, genug für die erwarteten 2 TB plus Wachstum, und verkraftet den Ausfall von zwei Platten. Software-RAID (mdadm) oder ZFS (RAID-Z2) wird im Phasenkonzept entschieden.
- **Internetanschluss prüfen:** Manche Anschlüsse nutzen DS-Lite (RFC 6333 [Q18]); dann gibt es keine eigene öffentliche IPv4-Adresse, und eine IPv4-Portfreigabe ist nicht möglich. Christian hat angegeben, dass sein Anschluss öffentlich erreichbar ist. Das wird vor dem Live-Gang konkret geprüft (öffentliche IPv4 und/oder IPv6, DynDNS-Fähigkeit des Routers).
- Domain, DynDNS, Portfreigabe nur für 443 (und 80 nur, falls für die Zertifikatsprüfung nötig).
- **Reverse Proxy und TLS:** Vorschlag nginx und certbot aus den Debian-Paketen (Entscheidung E2). Alternative: TLS und ACME direkt im eigenen Server, was eine Fremdkomponente spart, aber sicherheitskritischen Code selbst verantwortet.
- Härtung: Firewall, nur Schlüssel-Login per SSH aus dem Heimnetz, automatische Sicherheitsupdates, eigener Systemnutzer für den Dienst.
- **Updates:** Sicherheitsupdates automatisch; große Updates per Klick, mit automatischem Backup vorher und dokumentiertem Rückweg.
- **Monitoring:** Plattenzustand (SMART), RAID-Status, Speicherplatz, Zertifikatslaufzeit, Backup-Erfolg, Fehlerraten. Alarm per Push und Mail. Ein Server kann seinen eigenen Totalausfall nicht melden; deshalb ist ein externer „Totmannschalter“ nötig, der Alarm schlägt, wenn der Server sich nicht mehr meldet (Entscheidung E7).
- **USV** mit sauberem Herunterfahren bei langem Stromausfall.
- **Notfallhandbuch** auf Papier und digital, verständlich auch für Christians Freundin: Wer hat welche Schlüssel, wie stellt man aus dem Backup wieder her, wen ruft man an.
- **Datenschutz:** Die DSGVO gilt nicht für Verarbeitung durch natürliche Personen „zur Ausübung ausschließlich persönlicher oder familiärer Tätigkeiten“ (Art. 2 Abs. 2 lit. c DSGVO [Q19]). Die Schutzmaßnahmen werden trotzdem so umgesetzt, als würde sie gelten.

## 9. Fotoalben, Android-App, Smart Home – Phasen 6 bis 8 (Umriss)

**Phase 6 Fotoalben:** Upload von Handy und PC, Alben, Freigaben je Person, Vorschaubilder, Metadaten (Datum, Ort), Suche nach Datum. Offene Punkte für das Phasenkonzept: Bilddekodierung (HEIC vom iPhone, Videos) braucht etablierte Bibliotheken, die über die bisher freigegebenen Bereiche hinausgehen; das wird dort als Entscheidung vorgelegt. Vorschlag, der vom Grundsatz „alles offline“ abweicht und daher im Phasenkonzept zur Entscheidung vorgelegt wird: Offline-Zugriff bei Fotos nur für ausgewählte Alben, weil der Gerätespeicher begrenzt ist.

**Phase 7 Android-Sync-App:** Kotlin-App, die sich als Konto in Android einträgt und Termine (später auch Kontakte, Aufgaben) direkt über die Sync-API des Kerns mit dem Android-Kalenderspeicher abgleicht, mit derselben Konfliktlogik. Ersetzt DAVx⁵. Verteilung als selbst signierte APK, kein Play Store nötig.

**Phase 8 Smart Home:** Höchstes Risiko, weil Fehler physische Folgen haben (Heizung, Türen). Grundsätze: Steuerung nur lokal, keine Cloud der Hersteller, eigene Rechte für Steuern und Ansehen, alles protokolliert, sicherer Zustand bei Ausfall. Welche Funkstandards (z. B. Zigbee, Matter, WLAN-Geräte) und wie viel davon selbst entwickelt wird, ist offen; vollständige Funkprotokoll-Stacks selbst zu schreiben wäre sehr aufwendig und wird im Phasenkonzept ehrlich bewertet.

---

## 10. Qualität und Arbeitsweise

- **Ablauf je Phase:** Konzept als Pull Request → Selbstreview und Copilot Code Review, wiederholt bis keine Befunde mehr → Freigabe durch Christian → Umsetzung in kleinen Pull Requests → jeweils Tests, Selbstreview und Copilot Review bis keine Befunde mehr → Abschluss der Phase.
- **Review-Kriterien:** Korrektheit, Sicherheit, Verständlichkeit, Testabdeckung, Einhaltung der Grundsätze in diesem Dokument, englischer Code, i18n.
- **Copilot Code Review:** In Copilot Free nicht enthalten, in Copilot Pro enthalten [Q20]. Jeder Review verbraucht AI-Credits; GitHub schätzt die Kosten je Review auf etwa 0,05 bis 1 US-Dollar bei Aufwand „Lite“ und 0,25 bis 5 US-Dollar bei „Balanced“ [Q21]. Bei einem Konzept, das mehrfach reviewt wird, kommen also einige Dollar je Phase zusammen. Automatische Reviews bei neuen Pull Requests und neuen Pushes lassen sich in den persönlichen Copilot-Einstellungen einschalten [Q22]. Ob Copilot reine Markdown-Konzepte inhaltlich sinnvoll prüft, ist nicht dokumentiert und wird beim ersten Pull Request getestet.
- **CI (GitHub Actions):** Für private Repos ist das kostenlose Kontingent an Actions-Minuten begrenzt; die Pipeline wird deshalb sparsam gehalten. Build, Lint, Typprüfung, Tests, Abhängigkeitsprüfung, Prüfung auf vollständige Übersetzungen.
- **Abhängigkeiten:** möglichst wenige, Versionen fest im Lockfile, automatische Update-Pull-Requests (Dependabot) mit Review.
- **Entscheidungen** werden kurz als Entscheidungsprotokoll in `/docs/decisions` festgehalten.

## 11. Offene Entscheidungen für Christian

| Nr. | Frage | Vorschlag |
|---|---|---|
| E1 | Technologie-Stack | TypeScript (Node.js LTS) + PostgreSQL, UI-Framework im Kern-Konzept |
| E2 | TLS / Reverse Proxy | nginx + certbot aus Debian |
| E3 | Reihenfolge ab Phase 4 | Datensicherung → Live-Betrieb → Fotoalben → Android-App → Smart Home |
| E4 | Externe Kalender: Richtung | Zuerst nur einlesen, Zwei-Wege-Sync später |
| E5 | Commit-Messages und PR-Titel | Englisch, wie der Code |
| E6 | Copilot Pro | Wird für Copilot Code Review gebraucht (falls noch nicht vorhanden) |
| E7 | Externer Totmannschalter für Monitoring | Kleiner externer Dienst, der bei fehlendem Lebenszeichen alarmiert; Auswahl in Phase 5 |
| E8 | Festplattenverschlüsselung am Server | Ja; offen ist, wie der Server nach Stromausfall entsperrt wird (Passphrase vor Ort/per SSH oder TPM) |
| E9 | Fehlversuche bei der Anmeldung | Ansteigende Wartezeit plus Alarm statt harter Sperre |
| E10 | Maximale Offline-Dauer ohne Online-Anmeldung | 30 Tage |
| E11 | Zugang zum Backup-Schlüssel im Notfall | Versiegelter Ausdruck an einem im Notfallhandbuch genannten Ort |
| E12 | Ausnahme für Kalenderdaten im Kalender des Betriebssystems (CalDAV, Android-App) | Zulassen, nur mit Bildschirmsperre und Geräteverschlüsselung, sensible Kalender vom Sync ausschließbar |
| E13 | Schlüsselmodell der Datensicherung | Öffentlicher Schlüssel auf dem Server, privater Wiederherstellungsschlüssel nur bei Christian; Entschlüsselungstests manuell |
| E14 | Geräteschlüssel auf dem Server als zweiter Weg zum Offline-Datenschlüssel (für Passwortwechsel und Sperren von Geräten) | Einführen (3.4.3) |

---

## 12. Quellen

Alle Quellen abgerufen am 2026-11-08.

- [Q1] RFC 8291, Message Encryption for Web Push. https://www.rfc-editor.org/rfc/rfc8291
- [Q2] W3C, Web Authentication Level 3, Recommendation vom 25. August 2026. https://www.w3.org/TR/webauthn-3/
- [Q3] RFC 6238, TOTP: Time-Based One-Time Password Algorithm. https://www.rfc-editor.org/rfc/rfc6238
- [Q4] RFC 9106, Argon2 Memory-Hard Function, Abschnitt 4 (Parameterempfehlungen). https://www.rfc-editor.org/rfc/rfc9106
- [Q5] Apple Support, App-spezifische Passwörter (2FA erforderlich). https://support.apple.com/en-us/102654
- [Q6] Let's Encrypt, Certificate Lifetime Rationale and Plans. https://letsencrypt.org/docs/cert-lifetimes/
- [Q7] RFC 6578, Collection Synchronization for WebDAV. https://www.rfc-editor.org/rfc/rfc6578
- [Q8] RFC 4791, Calendaring Extensions to WebDAV (CalDAV). https://www.rfc-editor.org/rfc/rfc4791
- [Q9] MDN Browser Compatibility Data, SyncManager (Safari: nicht unterstützt). https://github.com/mdn/browser-compat-data/blob/main/api/SyncManager.json
- [Q10] RFC 8292, Voluntary Application Server Identification (VAPID) for Web Push. https://www.rfc-editor.org/rfc/rfc8292
- [Q11] WebKit Blog, Web Push for Web Apps on iOS and iPadOS. https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
- [Q12] WebKit Blog, Full Third-Party Cookie Blocking and More (7-Tage-Grenze, Home-Bildschirm-Apps). https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/
- [Q13] RFC 5545, Internet Calendaring and Scheduling Core Object Specification (iCalendar). https://www.rfc-editor.org/rfc/rfc5545
- [Q14] Nextcloud-Dokumentation, Synchronizing with iOS (Einrichtung eines CalDAV-Kontos im iOS-Kalender). https://docs.nextcloud.com/server/latest/user_manual/en/pim/sync_ios.html
- [Q15] DAVx⁵ FAQ. https://www.davx5.com/faq
- [Q16] Google, CalDAV API Developer's Guide. https://developers.google.com/workspace/calendar/caldav/v2/guide
- [Q17] Google, Using OAuth 2.0 to Access Google APIs, Abschnitt „Refresh token expiration“. https://developers.google.com/identity/protocols/oauth2
- [Q18] RFC 6333, Dual-Stack Lite Broadband Deployments Following IPv4 Exhaustion. https://www.rfc-editor.org/rfc/rfc6333
- [Q19] Verordnung (EU) 2016/679 (DSGVO), Art. 2 und Erwägungsgrund 18. Amtlicher Text: https://eur-lex.europa.eu/eli/reg/2016/679/oj; Wortlaut von Art. 2 Abs. 2 lit. c geprüft über https://dsgvo-gesetz.de/art-2-dsgvo/
- [Q20] GitHub, Copilot plans (Zeile „Pull request reviews in GitHub“). https://github.com/features/copilot/plans
- [Q21] GitHub Docs, About Copilot code review. https://docs.github.com/en/copilot/concepts/agents/code-review
- [Q22] GitHub Changelog, 2026-09-23: Copilot code review: more ways to request and configure reviews. https://github.blog/changelog/2026-09-23-copilot-code-review-more-ways-to-request-and-configure-reviews
- [Q23] RFC 9110, HTTP Semantics, Abschnitte 13.1.1 (If-Match) und 15.5.13 (412 Precondition Failed), https://www.rfc-editor.org/rfc/rfc9110; zusammengefasst bei MDN: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/412
