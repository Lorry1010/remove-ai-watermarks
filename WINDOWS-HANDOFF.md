# Windows-Handoff: Remove AI Watermarks

Stand: 14. Oktober 2026. Diese Datei übergibt die bisherige Recherche und die
Windows-Startschritte an die nächste Sitzung.

## Ausgangspunkt und Ziel

- Eigener Fork: <https://github.com/Lorry1010/remove-ai-watermarks>
- Upstream: <https://github.com/wiltodelta/remove-ai-watermarks>
- Basis dieser Recherche: `4241a01659de4fda6980ade15ba5a1a2836db345`, Paketversion
  `0.45.1`, Branch `main`.
- Die Vorbereitung fand auf einem Mac statt. Die eigentliche Bildverarbeitung
  soll auf dem Windows-Rechner mit NVIDIA-GPU laufen.
- Windows-Hardware laut Nutzer: **64 GB RAM**, **GeForce RTX 4070**. Zuvor wurde
  **4070 Ti** genannt; das genaue Modell und den verfügbaren VRAM auf Windows
  mit `nvidia-smi` prüfen. Beide regulären Karten haben 12 GB VRAM; die
  Ti-SUPER-Variante hätte 16 GB.
- Schwerpunkt: unsichtbare **Bild**wasserzeichen wie SynthID, insbesondere in
  Google-/Gemini-Bildern. Sichtbare Markierungen und Metadaten sind zusätzliche
  Funktionen.
- Gewünscht ist zunächst ein Handoff zum Klonen und Weiterarbeiten. Eine
  einfache lokale UI wurde als mögliche nächste Ausbaustufe besprochen; sie
  ist **noch nicht implementiert**, und das UI-Framework ist nicht entschieden.

Diese Übergabe fügt Dokumentation hinzu. Auf dem Windows-Rechner wurden noch
keine Installation, CUDA-Prüfung oder Bildverarbeitung durchgeführt. Die
Hardware-Einschätzung ist daher keine Bestätigung eines erfolgreichen Laufs.

## Was das vorhandene Projekt liefert

Das Repository enthält eine CLI und eine Python-Bibliothek, aber keine eigene
lokale GUI. Nach einem CLI-Lauf öffnet man die Ausgabedatei in einer normalen
Bildanzeige. Für eine grafische Bedienung gibt es separat
[ComfyUI-Nodes](https://github.com/wiltodelta/ComfyUI-remove-ai-watermarks).
ComfyUI ist keine Voraussetzung für den hier beschriebenen Weg.

Die drei Bildfunktionen unterscheiden sich:

| Funktion | Verfahren | Hardware |
| --- | --- | --- |
| Sichtbare Markierungen | Erkennen, maskieren und die Stelle auffüllen | CPU genügt |
| KI-Metadaten | Dateimetadaten wie C2PA, EXIF und XMP bearbeiten | CPU genügt |
| Unsichtbare Bildwasserzeichen | Bild durch Diffusionsmodelle regenerieren | NVIDIA/CUDA erforderlich |

SynthID wird bei diesem Verfahren durch veränderte Bildpixel gestört. Es wird
kein separat gespeicherter Wasserzeichen-Layer gelöscht. Gesichter, Schrift,
Farben und feine Details können sich verändern. Das Paket hat **keinen lokalen
SynthID-Pixeldetektor**; ein erfolgreicher Prozess oder ein unauffälliger
Metadatenbericht beweist deshalb keine vollständige SynthID-Entfernung.

Das aktuelle Projekt bietet `qwen-zimage`, `sdxl-zimage`, `chroma-zimage` und
`auto`. Für einen ersten Google-/Gemini-Versuch ist `sdxl-zimage` mit
CPU-Offload und zunächst 1024 Pixeln Arbeitsauflösung die Empfehlung aus dieser
Sitzung. Die benötigte Installations-Erweiterung heißt trotzdem `qwen-zimage`:
Sie bringt auch die gemeinsame Z-Image-Gesichtsstufe und die anderen benötigten
Diffusionsabhängigkeiten mit.

64 GB System-RAM helfen beim Offload, ersetzen aber keinen GPU-Speicher.
Die aktuelle SDXL-Implementierung unterstützt sequenziellen CPU-Offload.
Upstream dokumentiert einen globalen SDXL-Lauf auf einer 8-GB-GPU unter WSL2;
das ist kein Benchmark der gesamten Pipeline auf diesem Windows-PC.

## Repository bei Bedarf zuerst privat stellen

Bei der Vorbereitung war `Lorry1010/remove-ai-watermarks` ein **öffentlicher
Fork**, ohne eigene Child-Forks und mit einer von GitHub gemeldeten Größe unter
1 GB. Die Sichtbarkeit wurde in dieser Sitzung nicht geändert.

GitHub lässt die Sichtbarkeit eines Forks nicht unabhängig ändern. Für das
bestehende Repository ist der vorgesehene Weg:

1. GitHub-Repository öffnen, dann **Settings → General → Danger Zone**.
2. Falls verfügbar, **Leave fork network** wählen und die GitHub-Hinweise lesen.
3. Nach Abschluss beim nun eigenständigen Repository
   **Change repository visibility → Private** wählen.
4. Prüfen, dass GitHub tatsächlich **Private** anzeigt.

Das Loslösen ist dauerhaft. Git-Commits bleiben erhalten; GitHub beschreibt
den Verlust verknüpfter Repository-Metadaten, darunter Issues, PRs, Wikis,
Stars und Watcher. Falls diese Daten erhalten werden sollen oder der Button
fehlt, stattdessen eine separate private Kopie planen. Den vorhandenen Fork
dafür nicht ungeprüft löschen.

Bei gleichem Besitzer und Namen bleibt die Clone-URL gleich. Der Zugriff auf
ein privates Repository benötigt auch auf dem Windows-PC eine eigene
GitHub-Anmeldung. Die Mac-Anmeldung wird nicht übertragen.

Quellen: [Fork-Sichtbarkeit](https://docs.github.com/en/pull-requests/reference/forks),
[Fork loslösen](https://docs.github.com/en/pull-requests/how-tos/work-with-forks/detaching-a-fork),
[Sichtbarkeit ändern](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/managing-repository-settings/setting-repository-visibility).

## Auf Windows klonen

Die folgenden Befehle sind für **PowerShell**, jeweils einzeln ausführen und
Fehler vor dem nächsten Schritt beheben. Bereits vorhandene Werkzeuge müssen
nicht erneut installiert werden. `C:\Developer` ist ein Beispielpfad.

### 1. Werkzeuge installieren

```powershell
winget install --id Git.Git -e
winget install --id GitHub.cli -e
winget install --id astral-sh.uv -e
```

Danach PowerShell neu öffnen, damit der aktualisierte PATH verwendet wird:

```powershell
git --version
gh --version
uv --version
nvidia-smi
```

Einen aktuellen NVIDIA-Treiber verwenden. Falls `nvidia-smi` die GPU nicht
findet, erst das Treiberproblem beheben. Ein CUDA-fähiges PyTorch-Wheel und
ein passender Treiber sind nötig; das separate CUDA-Toolkit ist für fertige
PyTorch-Wheels normalerweise keine Voraussetzung.

### 2. GitHub-Anmeldung und Checkout

```powershell
gh auth login --hostname github.com --git-protocol https --web
gh auth setup-git
New-Item -ItemType Directory -Force -Path "C:\Developer" | Out-Null
gh repo clone Lorry1010/remove-ai-watermarks "C:\Developer\remove-ai-watermarks" -- --branch main
Set-Location "C:\Developer\remove-ai-watermarks"
git status --short --branch
Get-Content .\WINDOWS-HANDOFF.md
```

Falls der Checkout bereits existiert, stattdessen in den Ordner wechseln,
`git status --short --branch` prüfen und bei sauberem `main` mit
`git pull --ff-only origin main` aktualisieren. Keine bestehenden Änderungen
zurücksetzen. Bei einer separaten privaten Kopie die Repository-Adresse anpassen.

## CUDA-Umgebung aus dem lokalen Quellcode installieren

Für Weiterentwicklung muss **dieser Checkout** installiert werden. Eine
globale Installation mit `uv tool install remove-ai-watermarks` würde sonst
das veröffentlichte Paket ausführen und lokale Änderungen nicht verwenden.

Python 3.11 bis 3.14 sind im Projekt erlaubt; für den ersten Windows-Aufbau
verwenden wir Python 3.12 und eine eigene `.venv`:

```powershell
uv python install 3.12
uv venv --python 3.12 .venv
uv pip install --python .\.venv\Scripts\python.exe --torch-backend=auto --editable ".[qwen-zimage]"
uv pip check --python .\.venv\Scripts\python.exe
```

`--editable` bindet die Installation an den lokalen Quellcode.
`--torch-backend=auto` lässt uv anhand des installierten Treibers einen
passenden PyTorch-Backend-Index wählen. Ohne erkannte GPU kann uv eine
CPU-Version wählen; deshalb ist die folgende Prüfung verbindlich.

Dieser erste Windows-Weg verwendet `uv pip` und die Anforderungen aus
`pyproject.toml`, **keine vollständige Reproduktion von `uv.lock`**. Der
Lockfile verweist für Torch auf PyPI; nach der uv-Dokumentation sind die
Windows-PyPI-Wheels CPU-Wheels. Der entsprechende Kommentar in
`pyproject.toml` ist daher keine zuverlässige CUDA-Installationsanleitung.

Die hier aufgebaute Umgebung direkt über `.venv\Scripts` verwenden. Ein
späteres `uv sync` oder ein automatisch synchronisierendes `uv run` kann sie
wieder auf den PyPI-Stand bringen. Eine dauerhafte Windows-CUDA-Konfiguration
für den Lockfile wäre ein eigener Folgeschritt.

### CUDA und Quellcode-Verbindung prüfen

```powershell
.\.venv\Scripts\python.exe -c "import torch; print('Torch:', torch.__version__); print('CUDA runtime:', torch.version.cuda); print('CUDA available:', torch.cuda.is_available()); assert torch.cuda.is_available(), 'CUDA ist nicht nutzbar'; print('GPU:', torch.cuda.get_device_name(0)); print('VRAM GiB:', round(torch.cuda.get_device_properties(0).total_memory / 1024**3, 2)); print('GPU allocation:', torch.zeros(1, device='cuda').device)"
.\.venv\Scripts\python.exe -c "import remove_ai_watermarks as raiw; print('Version:', raiw.__version__); print('Source:', raiw.__file__)"
.\.venv\Scripts\remove-ai-watermarks.exe invisible --help
```

Erwartet: `CUDA available: True`, eine NVIDIA-GPU, erfolgreiche CUDA-Allokation
und ein `Source`-Pfad unter dem geklonten `src`-Verzeichnis. Keine schwere
Bildverarbeitung beginnen, solange diese Prüfungen fehlschlagen.

Falls CPU-Torch bereits installiert war, vor einer erneuten Prüfung gezielt
Torch und Torchvision aus dem automatisch gewählten Backend neu installieren:

```powershell
uv pip install --python .\.venv\Scripts\python.exe --torch-backend=auto --reinstall-package torch --reinstall-package torchvision torch torchvision
uv pip check --python .\.venv\Scripts\python.exe
```

Quellen: [uv unter Windows installieren](https://docs.astral.sh/uv/getting-started/installation/),
[uv und PyTorch-Backends](https://docs.astral.sh/uv/guides/integration/pytorch/),
[offizielle PyTorch-Installation](https://pytorch.org/get-started/locally/).

## Erster Bildversuch

Zunächst ein einzelnes eigenes Google-/Gemini-Bild verwenden. Das Original
behalten. Die Pfade im folgenden Beispiel vorher an die echte Datei anpassen.
`out/` ist bereits im Repository ignoriert.

```powershell
New-Item -ItemType Directory -Force -Path .\out | Out-Null
.\.venv\Scripts\remove-ai-watermarks.exe identify "C:\Bilder\bild.png"
.\.venv\Scripts\remove-ai-watermarks.exe invisible "C:\Bilder\bild.png" -o ".\out\bild_clean.png" --pipeline sdxl-zimage --vendor google --cpu-offload --max-resolution 1024
```

`--vendor google` nur bei bekannter Google-Herkunft verwenden. Es wählt die
Google-Stärke und lässt die Regeneration auch ohne lesbare lokale Provenienz
laufen. Ein zusätzliches `--force` ist dann nicht nötig. Ein stiller oder
unbekannter `identify`-Befund bedeutet nicht, dass SynthID fehlt.

Die Grenze `1024` verkleinert die lange Seite vor der Verarbeitung und stellt
danach die Ausgabegröße wieder her; das kostet Details. Sie ist eine
Startkonfiguration für den Funktionstest, keine empfohlene Endqualität für
jedes Bild. Nach einem erfolgreichen Versuch höhere Auflösungen oder
`--tile --max-resolution 0` separat prüfen.

Beim ersten Lauf werden Modellgewichte heruntergeladen und gecacht. Der
erste Lauf ist deshalb kein aussagekräftiger Geschwindigkeitstest. Nachher
Original und Ergebnis nebeneinander prüfen, besonders Gesichter und Schrift.
Eine Prüfung mit dem passenden Anbieter-Verifier ist eine separate Prüfung;
die CLI zertifiziert SynthID-Freiheit nicht.

Für die Fortsetzung notieren:

- Genaues GPU-Modell, Treiberversion, Torch-/CUDA-Version und Commit.
- Kommando, Laufzeit nach dem ersten Download und beobachteter Speicherbedarf.
- Ob die Bilddatei erzeugt wurde und welche sichtbaren Änderungen auffallen.
- Bei Fehlern den tatsächlichen Fehlertext; keine Zugangsdaten mitprotokollieren.

Die Paketversionen können lokal unter `out/` festgehalten werden:

```powershell
uv pip freeze --python .\.venv\Scripts\python.exe > .\out\windows-environment.txt
```

Bei nativen Windows-Abhängigkeitsfehlern zuerst den konkreten Fehler prüfen.
WSL2 mit CUDA ist ein möglicher Ausweichweg und wird in den Upstream-Messungen
verwendet; es wurde in dieser Sitzung nicht auf dem Zielrechner eingerichtet.

## Mögliche nächste Ausbaustufe: einfache lokale UI

Dieser Abschnitt ist ein Vorschlag für die Fortsetzung, kein bereits
implementierter oder abschließend abgestimmter Funktionsumfang.

Nach erfolgreichem Windows-CLI-Test wäre eine kleine lokale Browser-UI sinnvoll:

- Bild auswählen oder hineinziehen, mit Vorschau.
- Modus auswählen: unsichtbares Bildwasserzeichen, sichtbare Markierung oder
  nur Metadaten. Den Google-Modus nur bei bekannter Herkunft auswählen.
- Starten, laufenden Arbeitsschritt sehen und verständliche Fehler erhalten.
- Vorher/Nachher anzeigen und das Ergebnis separat speichern.
- CPU-Offload als geeignete Voreinstellung für die 12-GB-Karte; Auflösung und
  Pipeline unter erweiterten Einstellungen.
- Verarbeitung zunächst sequenziell; ein GPU-Auftrag zur Zeit.

Framework und Verpackung auf Windows sind noch offen. Die bestehende
Bildverarbeitung wiederverwenden. Für einen kleinen ersten Prototyp kann die
CLI als separater Prozess ohne Shell aufgerufen werden; das erlaubt auch ein
zuverlässiges Beenden eines laufenden Prozesses. Für spätere direkte
Integration existieren `InvisibleOptions`, `remove_all` und `remove_batch` in
der öffentlichen Python-API. Kein öffentliches `remove_invisible` für Bilder
erfinden: ein eigener Adapter muss zur tatsächlich vorhandenen API passen.

Die UI lokal auf `127.0.0.1` starten. Die Fertigmeldung soll bedeuten, dass eine
Ausgabedatei erzeugt wurde, nicht dass SynthID-Freiheit bewiesen wurde. Private
Beispielbilder, Modelle und Zugangsdaten gehören nicht in Git.

## Wichtige Dateien für die nächste Sitzung

| Datei | Wofür sie relevant ist |
| --- | --- |
| [CLAUDE.md](CLAUDE.md) | Projektkonventionen und Prüfvorgaben |
| [pyproject.toml](pyproject.toml) | Python-Version, Extras und CLI-Einstieg |
| [docs/installation.md](docs/installation.md) | Feature- und Modellabhängigkeiten |
| [docs/cli.md](docs/cli.md) | Gültige Befehle und Optionen |
| [docs/known-limitations.md](docs/known-limitations.md) | Speicher, Bildqualität und Detektorgrenzen |
| [docs/module-internals.md](docs/module-internals.md) | Architektur und Offload-Messungen |
| [src/remove_ai_watermarks/cli.py](src/remove_ai_watermarks/cli.py) | CLI-Adapter und Statusausgabe |
| [src/remove_ai_watermarks/api.py](src/remove_ai_watermarks/api.py) | Öffentliche API und Ergebnisobjekte |
| [src/remove_ai_watermarks/_internal/sdxl_zimage_pipeline.py](src/remove_ai_watermarks/_internal/sdxl_zimage_pipeline.py) | SDXL-Lader und sequenzieller Offload |
| [src/remove_ai_watermarks/_internal/two_stage_pipeline.py](src/remove_ai_watermarks/_internal/two_stage_pipeline.py) | Gemeinsame Gesichtsstufe und GPU-Speichermanagement |

CLI-Optionen und Offload-Verhalten wurden auf dem genannten Basis-Commit im
Quellcode geprüft. Die Windows-Kommandos wurden auf dem Mac nicht ausgeführt.
Eine Dokumentationsprüfung ersetzt keinen CUDA-Test auf dem Zielrechner.

## Starttext für Codex auf dem Windows-PC

```text
Lies WINDOWS-HANDOFF.md und die Projektkonventionen. Wir setzen die auf dem
Mac vorbereitete Arbeit in meinem eigenen remove-ai-watermarks-Repository
auf diesem Windows-PC fort. Er hat 64 GB RAM und eine RTX 4070 bzw. 4070 Ti;
prüfe das genaue GPU-Modell hier. Untersuche Repository-Dateien direkt lokal.

Prüfe Git-Status und Remote, installiere bei Bedarf eine lokale editierbare
Python-3.12-Umgebung mit CUDA-PyTorch und verifiziere die GPU-Nutzung. Teste
danach ein einzelnes von mir ausgewähltes eigenes Google-/Gemini-Bild mit
sdxl-zimage und CPU-Offload. Frage nach dem Bildpfad, falls noch kein Testbild
benannt ist. Bewahre das Original und dokumentiere das tatsächliche Ergebnis.

Als mögliche nächste Ausbaustufe haben wir eine einfache lokale UI mit
Bildauswahl, Start, Fortschritt und Vorher/Nachher-Vergleich besprochen.
Stimme diesen nächsten Schritt mit mir ab, nachdem der CLI-Test funktioniert.
ComfyUI ist für den direkten lokalen Weg nicht erforderlich.
```
