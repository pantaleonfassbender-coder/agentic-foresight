# Agentic Foresight — Deploy-Anleitung

Website für agentische Foresight-Recherchen (Deutsch/Englisch), umgesetzt nach einer Idee von:
Haugk, S. & Leyh, C. (2026): *Agentic Foresight: Potenziale autonomer KI-Agenten
für die strategische Vorausschau in Unternehmen.* HMD Praxis der Wirtschaftsinformatik.
doi:10.1365/s40702-026-01318-4

## Deploy per Netlify Drop (kein Build nötig)

1. https://app.netlify.com/drop öffnen (bei Netlify eingeloggt sein).
2. Diesen kompletten Ordner (`agentic-foresight-site`) per Drag & Drop in die Drop-Zone ziehen.
3. Fertig. Die Seite ist sofort unter der zugeteilten `*.netlify.app`-URL erreichbar.

**Kein API-Key nötig:** Die Serverless Function nutzt das Netlify AI Gateway —
Netlify stellt `ANTHROPIC_API_KEY` und `ANTHROPIC_BASE_URL` automatisch bereit
und rechnet die KI-Nutzung in Netlify-Credits ab. Das Gateway wird mit dem ersten
Production-Deploy aktiv; falls die allererste Recherche direkt nach dem Deploy
einen Key-Fehler meldet, kurz warten bzw. einmal neu deployen.

## Falls die Function per Drop nicht mitkommt (Fallback)

Die Netlify CLI ist installiert und eingeloggt. Im Ordner ausführen:

    netlify deploy --prod --dir . 

(bzw. beim ersten Mal `netlify init` / Site auswählen).

## Architektur

- `index.html` — komplette Oberfläche (DE/EN), orchestriert die drei Agenten-Phasen
  clientseitig und zeigt jede Websuche und jeden Zwischenschritt live an.
  Scout- und Validator-Phase laufen in mehreren kurzen Aufrufen statt in einem
  langen: Der Scout (P1) recherchiert perspektivenweise (Technologie/Forschung,
  Markt/Wettbewerb, bei „Tief“ zusätzlich Regulierung/Lieferketten/Gesellschaft),
  der Validator (P2) prüft in Paketen von zwei Signalen. Jeder einzelne Aufruf
  bleibt so deutlich unter dem 60-s-Limit einer Netlify-Function. Abgeschlossene
  Durchgänge und bereits bewertete Signale werden gemerkt — ein erneuter Versuch
  holt nur den offenen Teil nach. Bricht ein Durchgang doch einmal ab, werden die
  bis dahin vollständig übertragenen Signale aus dem Rohtext gerettet.
  Die Signale aller Durchgänge werden zusammengeführt und über die Quell-URL
  entdoppelt; die laufenden Durchgänge sehen die bereits gefundenen Signale und
  suchen gezielt nach anderen.
  Die Bewertungen werden in P2 als eigene Signal-Liste angezeigt (eskalierte zuerst);
  dort lässt sich jedes Signal per Häkchen von der Synthese ausschließen.
  Auch der Bericht (P3) entsteht in Segmenten: Reicht die Antwortlänge eines
  Aufrufs nicht bis zum Schluss (`stop_reason: max_tokens`) oder bricht ein
  Segment am Zeitbudget ab, schickt der Client den bisher geschriebenen Text
  zurück und lässt den Bericht fortsetzen — bis zu fünf Segmente. Bis dahin
  endeten längere Berichte (typisch bei „Tief“) ohne jede Fehlermeldung mitten
  im Satz: Der Stream endete regulär, nur eben am Token-Limit, und der Client
  nahm den abgeschnittenen Text als fertigen Bericht. Angesetzt wird an der
  letzten vollständigen Zeile — die angefangene Zeile am Abbruchpunkt wird
  verworfen und neu geschrieben.
  Der Bericht kann als Markdown oder als PDF (über den Druckdialog, inkl. Anhang
  mit allen geprüften Signalen) heruntergeladen werden — ohne Zusatzbibliothek.
- `netlify/functions/agent.mjs` — eine Function für alle drei Phasen
  (`scout` / `validator` / `synthesis`). Ruft die Anthropic Messages API
  (Modell: claude-sonnet-5, per Env `FORESIGHT_MODEL` änderbar) mit dem
  serverseitigen Web-Search-Tool auf und streamt die Antwort als SSE an den Browser
  durch. Der Durchgangs-Plan des Scouts (Perspektiven und Suchbudget) steht
  serverseitig und wird vom Client über `GET /api/agent?plan=scout&depth=…`
  abgefragt, damit Plan und Prompt nicht auseinanderlaufen. Ein eigenes Zeitbudget
  (55 s, `FORESIGHT_STREAM_BUDGET_MS`) bricht kurz vor dem harten 60-s-Limit ab und
  sendet ein lesbares SSE-`error`-Event, statt den Stream einfach abzuschneiden —
  sonst zeigt der Browser nur „network error“. Für die Synthese nimmt die Function
  den bereits geschriebenen Berichtsteil (`previous`, Segmentlänge über
  `FORESIGHT_SYNTHESIS_TOKENS`) entgegen und lässt das Modell ab der nächsten
  Zeile weiterschreiben. Der Text wird dabei in der User-Nachricht übergeben,
  nicht als Assistant-Prefill: Das Modell lehnt eine Konversation ab, die nicht
  mit einer User-Nachricht endet.

## Verhalten bei Störungen des AI Gateways

- Auf die Antwort-Header der Anthropic-API wird bis zu 30 s gewartet
  (`FORESIGHT_HEADER_TIMEOUT_MS`); die Wartezeit wird zusätzlich auf das
  verbleibende Gesamtbudget begrenzt. Gemessen liefert das Gateway das erste Byte
  nach ca. 2 s (ohne Websuche) bzw. 4–5 s (mit Websuche) — die frühere Grenze von
  20 s hatte bei Lastspitzen zu wenig Reserve.
- Kurzlebige Störungen (Verbindungsfehler, 429/5xx/529) wiederholt die Function
  einmal selbst, sofern danach noch genug Zeit für den Stream bleibt; ein
  `Retry-After`-Header wird berücksichtigt.
- Jede Fehlerantwort enthält neben dem Text die Felder `code` und `retryable`.
  Der Client entscheidet daran, ob er die Phase automatisch ein zweites Mal
  startet — vorher wurde das am Fehlertext geraten, wodurch ausgerechnet der
  Timeout-Fall ohne automatischen zweiten Versuch blieb. Dauerhafte Fehler
  (fehlender Key, ungültiges Modell, fehlendes Thema) werden weiterhin nie
  automatisch wiederholt.

- Keine npm-Abhängigkeiten, kein Build-Schritt — bewusst, damit Drag & Drop reicht.

## Kosten / Limits

- Abrechnung in Netlify-Credits über das AI Gateway (180 Credits ≈ 1 USD).
- Scout: „Standard“ = 2 Durchgänge à 3 Suchen (≈ 6 Suchen, ≈ 6 Signale);
  „Tief“ = 3 Durchgänge à 3 Suchen (≈ 9 Suchen, ≈ 9 Signale). Die Recherchetiefe
  ist damit eher höher als in der früheren Ein-Aufruf-Variante (5 bzw. 8 Suchen).
- Validator: bis zu 2 (Standard) bzw. 3 (Tief) Suchen pro Signal, maximal
  4 bzw. 6 pro Paket. Die Gesamtzahl der Suchen wächst damit mit der Anzahl der
  geprüften Signale — dafür läuft die Phase zuverlässig durch.
- Websuchen und lange Antworten kosten entsprechend mehr Credits.
- Synthese: 5.000 Token je Segment (`FORESIGHT_SYNTHESIS_TOKENS`). Ein kurzer
  Bericht ist damit nach einem Aufruf fertig; ein langer kostet je Fortsetzung
  einen weiteren Aufruf, in dem der bisherige Text erneut als Eingabe mitläuft.
- Gemessene Laufzeit je Scout-Durchgang: ca. 25–36 s bei „Tief“ — mit klarem
  Abstand zum 55-s-Budget und zum harten 60-s-Limit der Function.
