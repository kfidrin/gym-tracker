# Setup: GitHub Pages + Google Drive

Einmalig, ca. 20 Minuten.

## 1. GitHub-Repo und Pages

1. Neues Repo anlegen, z. B. `gym-tracker`.
   - Für kostenloses GitHub Pages muss das Repo **öffentlich** sein.
   - Das ist unkritisch: Im Repo liegt nur Code, deine Trainingsdaten liegen in
     deinem Drive.
2. Die Dateien `SPEC.md`, `CLAUDE.md` und `SETUP.md` ins Repo legen, dann
   Claude Code im Repo starten.
3. Sobald `index.html` existiert: Repo → *Settings → Pages → Source: Deploy
   from a branch → main / (root)*.
4. Die App-URL lautet dann `https://<dein-github-name>.github.io/gym-tracker/`.

## 2. Google Cloud: OAuth-Client für Drive

Auf https://console.cloud.google.com, mit deinem Google-Konto:

1. **Projekt anlegen**: oben in der Projektauswahl auf *Neues Projekt*,
   Name z. B. „Gym Tracker“.
2. **Drive-API aktivieren**: *APIs & Dienste → Bibliothek →* „Google Drive
   API“ suchen *→ Aktivieren*.
3. **OAuth-Zustimmungsbildschirm** (heißt in der neuen Oberfläche *Google
   Auth Platform*):
   - App-Name „Gym Tracker“, als Support-E-Mail deine Gmail-Adresse.
   - Zielgruppe: **Extern**.
   - Unter *Zielgruppe / Testnutzer* deine eigene Gmail-Adresse hinzufügen.
   - Die App bleibt im Status **„Testen“**. Eine Verifizierung ist nicht
     nötig.
4. **Datenzugriff / Bereiche**: den Bereich
   `https://www.googleapis.com/auth/drive.file` hinzufügen.
5. **Client erstellen** (*Clients* bzw. *Anmeldedaten → OAuth-Client-ID*):
   - Typ: **Webanwendung**.
   - *Autorisierte JavaScript-Quellen*:
     - `https://<dein-github-name>.github.io` (ohne Pfad, ohne Slash am Ende)
     - `http://localhost:8000` (zum lokalen Testen)
   - *Weiterleitungs-URIs*: leer lassen.
6. Die **Client-ID** kopieren (endet auf `.apps.googleusercontent.com`) und
   Claude Code geben. Sie kommt als `GOOGLE_CLIENT_ID` in `index.html`. Die
   Client-ID ist nicht geheim, ein Client-Secret wird nicht gebraucht.

Beim ersten Verbinden zeigt Google „Diese App wurde nicht verifiziert“. Klick
dann auf *Erweitert → Weiter zu Gym Tracker*. Das ist normal für eigene Apps
im Testmodus.

## 3. Auf dem Handy installieren

1. Die App-URL in Chrome auf Android öffnen.
2. Menü (⋮) → **App installieren** bzw. *Zum Startbildschirm hinzufügen*.
3. In der App unter *Settings → Connect* mit Google verbinden.

Am Laptop geht es genauso in Chrome, über das Installations-Symbol in der
Adressleiste.

## Hinweise

- Die Datei liegt in deinem Drive unter **Gym Tracker/gym.json**. Tägliche
  Backups liegen daneben als `gym-backup-YYYY-MM-DD.json`.
- Wegen der Berechtigung `drive.file` sieht die App nur die Dateien, die sie
  selbst angelegt hat. Wenn du später einen neuen OAuth-Client anlegst, findet
  die App die alte Datei nicht mehr. Dann einfach per Export/Import umziehen.
- Ein Google-Login hält etwa eine Stunde. Danach zeigt die App „reconnect“, ein
  Tippen genügt. Bis dahin wird lokal gespeichert, und es geht nichts verloren.
