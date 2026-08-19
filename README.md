# Agentic Foresight — Scout · Validator · Synthese

Eine Website für agentische Foresight-Recherchen: Zu einem Thema sucht ein Scout im offenen Web nach schwachen Signalen, ein Validator prüft jedes einzelne davon gegen unabhängige Quellen, und eine Synthese schreibt daraus einen Bericht mit Belegen. Jede Websuche und jeder Zwischenschritt ist live mitzulesen.

Oberfläche auf **Deutsch und Englisch**. Eine einzige statische Seite und eine Serverless-Function — kein Build-Schritt, keine npm-Abhängigkeiten, kein API-Schlüssel nötig.

Umgesetzt nach einer Idee von: Haugk, S. & Leyh, C. (2026): *Agentic Foresight: Potenziale autonomer KI-Agenten für die strategische Vorausschau in Unternehmen.* HMD Praxis der Wirtschaftsinformatik. [doi:10.1365/s40702-026-01318-4](https://doi.org/10.1365/s40702-026-01318-4)

---

## Die drei Phasen

| Phase | Was passiert | Websuche |
|---|---|---|
| **P1 · Scout** | Recherchiert perspektivenweise nach schwachen Signalen: Technologie/Forschung, Markt/Wettbewerb, bei „Tief" zusätzlich Regulierung, Lieferketten, Gesellschaft. Die laufenden Durchgänge sehen die bereits gefundenen Signale und suchen gezielt nach anderen; die Ergebnisse werden über die Quell-URL entdoppelt. | ja |
| **P2 · Validator** | Prüft die Signale in Paketen von zwei gegen unabhängige Quellen und eskaliert, was sich nicht hält. Jedes Signal lässt sich per Häkchen von der Synthese ausschließen. | ja |
| **P3 · Synthese** | Schreibt den Bericht aus den verbliebenen Signalen — als Markdown oder als PDF (über den Druckdialog, inklusive Anhang mit allen geprüften Signalen). | nein |

Zwischen den Phasen sitzt optional ein **Human-in-the-Loop-Gate**: Der Durchgang hält an und wartet auf Freigabe, statt selbsttätig weiterzulaufen.

**Recherchetiefe:** „Standard" = 2 Scout-Durchgänge à 3 Suchen; „Tief" = 3 Durchgänge à 3 Suchen, plus mehr Prüfsuchen je Signal.

---

## Warum die Phasen in vielen kurzen Aufrufen laufen

Eine Netlify-Function bricht nach 60 Sekunden hart ab. Der Entwurf arbeitet konsequent darum herum, und das ist der Teil, den man kennen sollte, bevor man etwas daran ändert:

- **Scout und Validator laufen stückweise** — perspektivenweise beziehungsweise in Paketen von zwei Signalen. Jeder einzelne Aufruf bleibt damit deutlich unter dem Limit. Gemessen: etwa 25–36 s je Scout-Durchgang bei „Tief".
- **Abgeschlossene Teile werden gemerkt.** Ein erneuter Versuch holt nur den offenen Teil nach. Bricht ein Durchgang doch ab, werden die bis dahin vollständig übertragenen Signale aus dem Rohtext gerettet.
- **Auch der Bericht entsteht in Segmenten.** Reicht ein Aufruf nicht bis zum Schluss (`stop_reason: max_tokens`), schickt der Client den bisher geschriebenen Text zurück und lässt fortsetzen — bis zu fünf Segmente, angesetzt an der letzten vollständigen Zeile. Vorher endeten längere Berichte ohne jede Fehlermeldung mitten im Satz: Der Stream endete regulär, nur eben am Token-Limit.
- **Ein eigenes Zeitbudget von 55 s** bricht kurz vor dem harten Limit ab und sendet ein lesbares SSE-`error`-Event, statt den Stream einfach abzuschneiden — sonst zeigt der Browser nur „network error".
- **Fehler tragen `code` und `retryable`.** Der Client entscheidet daran, ob er eine Phase automatisch wiederholt, statt es am Fehlertext zu raten. Dauerhafte Fehler — fehlender Schlüssel, ungültiges Modell, fehlendes Thema — werden nie automatisch wiederholt.
- **Der Scout-Plan steht serverseitig** und wird vom Client über `GET /api/agent?plan=scout&depth=…` abgefragt, damit Plan und Prompt nicht auseinanderlaufen können.

---

## Aufbau

```
index.html              komplette Oberfläche (DE/EN), orchestriert die drei Phasen clientseitig
netlify/functions/
  agent.mjs             eine Function für alle drei Phasen (scout / validator / synthesis);
                        ruft die Anthropic Messages API mit dem serverseitigen Web-Search-Tool
                        auf und streamt die Antwort als SSE an den Browser durch
netlify.toml            Publish-Verzeichnis, Function-Verzeichnis, Sicherheits-Header
ANLEITUNG.md            Deploy-Anleitung und Betriebsdetails
```

Keine npm-Abhängigkeiten und kein Build-Schritt — bewusst, damit Drag & Drop auf Netlify Drop genügt.

---

## Konfiguration

**Kein eigener API-Schlüssel nötig.** Die Function nutzt Netlifys **AI Gateway**; Netlify stellt `ANTHROPIC_API_KEY` und `ANTHROPIC_BASE_URL` automatisch bereit und rechnet über Netlify-Credits ab. Das Gateway wird mit dem ersten Production-Deploy aktiv.

| Variable | Vorgabe | Zweck |
|---|---|---|
| `FORESIGHT_MODEL` | `claude-sonnet-5` | Modell für alle drei Phasen |
| `FORESIGHT_STREAM_BUDGET_MS` | `55000` | Eigenes Zeitbudget vor dem harten 60-s-Limit |
| `FORESIGHT_HEADER_TIMEOUT_MS` | `30000` | Wartezeit auf die Antwort-Header der API |
| `FORESIGHT_SYNTHESIS_TOKENS` | `5000` | Segmentlänge des Berichts |

Kurzlebige Störungen (Verbindungsfehler, 429/5xx/529) wiederholt die Function einmal selbst, sofern danach noch genug Zeit für den Stream bleibt; ein `Retry-After`-Header wird berücksichtigt.

---

## Deployment

Per Netlify Drop, ohne Build: den kompletten Ordner auf <https://app.netlify.com/drop> ziehen. Die ausführliche Anleitung samt Fallback über die Netlify CLI steht in [ANLEITUNG.md](ANLEITUNG.md).

---

## Kosten

Abgerechnet wird in Netlify-Credits über das AI Gateway (180 Credits ≈ 1 USD). Die Größenordnung hängt an der Zahl der Websuchen: Scout etwa 6 („Standard") bis 9 Suchen („Tief"), Validator bis zu 2 beziehungsweise 3 Suchen je geprüftem Signal. Ein langer Bericht kostet je Fortsetzungssegment einen weiteren Aufruf, in dem der bisherige Text erneut als Eingabe mitläuft. [ANLEITUNG.md](ANLEITUNG.md) rechnet das im Detail auf.

---

## Grenzen

Das Werkzeug recherchiert im offenen Web und gibt wieder, was es dort findet. Der Validator prüft Signale gegen unabhängige Quellen und markiert, was sich nicht hält — er macht aus einer schwachen Quelle aber keine starke. Der Bericht ist eine belegte Zusammenstellung, keine geprüfte Prognose; die Belege sind zum Nachlesen da.

---

## Lizenz

Keine Lizenzdatei enthalten. Alle Rechte beim Autor, solange keine Lizenz ergänzt wird.

© 2026 — Dr. Pantaleon Fassbender
