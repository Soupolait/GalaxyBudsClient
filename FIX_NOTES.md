# Notes locales : fix « BlueZ timed out » (issue #640) + autostart

> Document de travail rédigé par une IA (Mistral Vibe, Mistral AI) le 2026-10-18,
> à la demande de Loïc. Fichier volontairement non commité / non suivi par git :
> il décrit la configuration locale (service systemd) en plus du patch de code.
> Le patch de code lui-même est commité : `6becfa74`.

## 1. Diagnostic (résumé)

Deux modes d'échec distincts au démarrage, observés dans `journalctl` sur 8 boots :

1. **Timeout BlueZ** (issue #640) : `le-connection-abort-by-local` répété,
   puis `Gave up after 5 attempts. Timed out` dans
   `GalaxyBudsClient.Platform.Linux/BluetoothService.cs`.
   Cause : les écouteurs ne sont pas en état d'accepter une connexion LE
   (dans le boîtier, endormis, ou connectés au téléphone — pas de multipoint
   sur les Buds4 Pro). Ce n'est PAS « BlueZ pas prêt » : bluetoothd répond
   à chaque tentative ; c'est l'établissement radio BlueZ→Buds qui échoue.
   Réessayer plus de fois n'y change rien tant que les Buds sont injoignables.

2. **Erreur résiduelle** : une fois l'abandon prononcé, l'app n'écoutait plus
   BlueZ. Quand les Buds devenaient disponibles (connexion via GNOME, mise
   à l'oreille), l'app ne s'en rendait pas compte : le profil SMEP n'était
   jamais attaché, l'erreur restait à l'écran, et un clic manuel sur
   « Connecter » fonctionnait instantanément (raccourci « déjà connecté »).

Il y avait aussi un crash au login (icône tray / extension AppIndicator GNOME,
exception Avalonia `DBusTrayIconImpl.CreateTrayIcon`), contourné côté système
voir §3 — non corrigé dans le code à ce jour.

## 2. Patch de code (commit 6becfa74)

Fichier : `GalaxyBudsClient.Platform.Linux/BluetoothService.cs` (+55 lignes).

- Nouveau watcher `RecoveryWatcher` (abonnement one-shot à la propriété D-Bus
  `Connected` du device BlueZ).
- Armé dans les deux chemins d'abandon de `ConnectAsync` :
  boucle des 5 tentatives, et timeout d'attente de la propriété `Connected`.
- Déclenchement : `Connected=true` (connexion initiée par le bureau, ou
  auto-connexion des Buds) → relance automatique de `ConnectAsync`, qui
  profite du raccourci « déjà connecté » pour attacher le profil SMEP en ~1 s.
- Gardes : effacé à l'entrée de tout nouveau cycle de connexion (un clic
  manuel le remplace) ; pas armé si le watchdog post-connexion est déjà
  actif (invariant « un seul abonnement à la fois » préservé) ; verrou
  anti-réentrance contre les recouvrements.

### Limites honnêtes
- **C'est un band-aid** sur le symptôme de l'issue #640, pas un fix des causes
  racines : les abandons `le-connection-abort-by-local` côté kernel et le
  timeout BlueZ de ~50 s par tentative restent inchangés.
- **Déclenché par évènement uniquement** : si les Buds redeviennent
  joignables mais que rien n'initie de connexion, aucune relance n'a lieu.
- **Coût du watcher** : un abonnement D-Bus persistant (signal
  `PropertiesChanged`) tant que l'app est en état de récupération.
  Pas de polling, pas de timer : sur la batterie du PC l'impact est
  négligeable, mais l'abonnement existe et consomme un minimum.
- Comportement inchangé sur Windows/macOS (backend Linux uniquement).

### Tests réalisés (CachyOS, GNOME 51, BlueZ/PipeWire, Buds4 Pro)
1. Buds injoignables → 5 échecs → watcher armé (ligne de log
   `Retrying automatically once the device becomes available`).
2. Buds laissées sur le téléphone → silence total, aucune boucle de retry.
3. Connexion manuelle via GNOME (sans toucher l'app) →
   `Device became available. Retrying connection...` puis
   `Connected to profile` ~1 s après, app pleinement fonctionnelle.
4. Double connexion manuelle concurrente → absorbée par les gardes
   pré-existantes (`Already connecting`), aucun crash.

## 3. Service systemd (contournement autostart)

Fichier : `~/.config/systemd/user/galaxybudsclient.service`
(remplace l'entrée `~/.config/autostart/galaxybudsclient.desktop`, supprimée).

Points importants de la config locale :

- `WantedBy=graphical-session.target` : démarre au login graphique.
- `ExecStartPre=/usr/bin/sleep 25` (actuellement réduit à 1 pour les tests) :
  laisse bluetoothd/wireplumber et l'extension AppIndicator s'initialiser.
  **Piège CachyOS** : `/usr/lib/systemd/user.conf.d/00-timeout.conf` impose
  `DefaultTimeoutStartSec=15s` aux unités utilisateur → sans le
  `TimeoutStartSec=45s` explicite dans l'unité, le sleep 25 est tué avant
  la fin et le service échoue en boucle (« start-pre operation timed out »).
- `Restart=on-failure` + `RestartSec=10` : relance l'app si elle crash
  (course de l'icône tray au login) ; l'extension est alors prête et la
  relance réussit.
- `StartLimitIntervalSec=120` / `StartLimitBurst=4` (section `[Unit]`) :
  arrêt définitif après 4 échecs en 2 min (les redémarrages espacés de
  >10 s contournaient la limite par défaut).
- `ExecStart` pointe actuellement sur le binaire local de test :
  `/home/loic/Téléchargements/dev/GalaxyBudsClient/bin_linux64/GalaxyBudsClient`
  (à remettre sur `/usr/bin/galaxybudsclient` quand une release/paquet
  officiel inclura le fix).

### Build local utilisé
```bash
dotnet publish -f net10.0 -r linux-x64 -o bin_linux64 -c Release \
  -p:PublishSingleFile=true --self-contained true \
  GalaxyBudsClient/GalaxyBudsClient.csproj
```
Note : `global.json` épingle le SDK 10.0.300 ; avec le SDK 10.0.113 installé,
il faut adapter temporairement la version puis restaurer le fichier
(le build du 2026-10-18 a été fait ainsi, `global.json` est revenu à l'état git).

## 4. Pour commenter l'issue #640

Éléments utiles à y mentionner :
- L'erreur vient de la boucle de retry de `BluetoothService.cs` ; les tentatives
  échouent au niveau radio (`le-connection-abort-by-local`), pas parce que
  BlueZ est indisponible.
- Après l'abandon, l'app n'écoute aucun signal BlueZ : une connexion ultérieure
  des écouteurs n'est jamais détectée, d'où l'erreur « qui reste ».
- Patch proposé (commit 6becfa74, backend Linux) : watcher one-shot sur
  `Connected` + relance auto. Testé matériellement (3 scénarios ci-dessus).
- Pistes non couvertes : retry sur évènement « device détecté pendant un scan »
  plutôt que `Connected=true` seulement ; interrogation de l'état
  (`GetConnectedAsync`) périodique légère.

## 5. Pour revenir en arrière

- Code : `git revert 6becfa74` (ou `git checkout 754b6fdf -- GalaxyBudsClient.Platform.Linux/BluetoothService.cs` avant le commit de revert).
- Service : rétablir `ExecStart=/usr/bin/galaxybudsclient /StartMinimized`,
  remettre `ExecStartPre=/usr/bin/sleep 25` si souhaité, puis
  `systemctl --user daemon-reload && systemctl --user restart galaxybudsclient.service`.
- Option d'origine : recréer `~/.config/autostart/galaxybudsclient.desktop`
  avec `Exec=/usr/bin/galaxybudsclient /StartMinimized`, puis
  `systemctl --user disable --now galaxybudsclient.service`.
