# 0001 – Entscheidungen aus dem Gesamtkonzept

Datum: 2026-11-08 · Status: angenommen

Christian hat das Gesamtkonzept (`docs/concept/overall-concept.md`, PR #1) mit „Passt, merge und fahre fort“ freigegeben. Die offenen Entscheidungen aus Abschnitt 11 gelten damit mit der jeweils empfohlenen Option als angenommen. Diese Lesart wurde Christian mitgeteilt; ändert er eine Entscheidung, wird sie hier mit Datum nachgetragen.

| Nr. | Entscheidung |
|---|---|
| E1 | TypeScript auf Node.js (aktuelle LTS) und PostgreSQL; UI-Framework wird im Kern-Konzept gewählt. |
| E2 | nginx und certbot aus den Debian-Paketen als Reverse Proxy und für TLS. |
| E3 | Reihenfolge ab Phase 4: Datensicherung → Live-Betrieb → Fotoalben → Android-App → Smart Home. |
| E4 | Externe Kalender zuerst nur einlesen, Zwei-Wege-Sync später. |
| E5 | Commit-Messages und PR-Titel auf Englisch. |
| E6 | Copilot Pro für Copilot Code Review (vorhanden, Reviews laufen). |
| E7 | Externer Totmannschalter für das Monitoring; Auswahl in Phase 5. |
| E8 | Festplattenverschlüsselung am Server; Entsperren nach Stromausfall wird in Phase 5 entschieden. |
| E9 | Ansteigende Wartezeit plus Alarm statt harter Kontosperre. |
| E10 | Maximale Offline-Dauer ohne Online-Anmeldung: 30 Tage. |
| E11 | Backup-Schlüssel im Notfall: versiegelter Ausdruck an einem im Notfallhandbuch genannten Ort. |
| E12 | Ausnahme für Kalenderdaten im Kalender des Betriebssystems, nur mit Bildschirmsperre und Geräteverschlüsselung; sensible Kalender vom Sync ausschließbar. |
| E13 | Backup mit öffentlichem Schlüssel auf dem Server, privater Wiederherstellungsschlüssel nur bei Christian; Entschlüsselungstests manuell. |
| E14 | Geräteschlüssel auf dem Server als zweiter Weg zum Offline-Datenschlüssel. |
