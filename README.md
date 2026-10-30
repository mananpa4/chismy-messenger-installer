# chismy-messenger-installer

Repo de **distribución y actualizaciones** de las apps Chismy (webview) y
Chismy Messenger (Flutter). Inspirada en el sistema de
[CompactFiles-Installer](https://github.com/mananpa4/CompactFiles-Installer):
un manifiesto `version.json` + assets en la carpeta `releases/` publicados como
GitHub Release. Las apps consultan el manifiesto al arrancar y ofrecen
instalar la versión nueva.

## Manifiesto (`version.json`)

```json
{
  "chismyMessenger": {
    "latest":            { "major": 1, "minor": 0, "patch": 0, "preRelease": "", "preReleaseMinor": 0 },
    "latestNonPreRelease": { "major": 1, "minor": 0, "patch": 0, "preRelease": "", "preReleaseMinor": 0 },
    "assets": {
      "x64":     "ChismyMessengerSetup.exe",
      "android": "ChismyMessenger.apk"
    }
  },
  "chismyApp": { ... "assets": { "x64": "ChismyAppSetup.exe", "android": "ChismyApp.apk" } }
}
```

- `latest`: última versión publicada (puede ser prerelease: `preRelease: "beta"`).
- `latestNonPreRelease`: última versión estable (los clientes estables pueden
  optar por ella).
- `assets`: **nombres fijos** de los archivos de cada release. Los clientes
  los descargan de
  `https://github.com/mananpa4/chismy-messenger-installer/releases/latest/download/<asset>`.
  Los nombres fijos permiten que el badge de la web y los clientes no cambien
  con cada versión.

URL del manifiesto para los clientes:

```
https://raw.githubusercontent.com/mananpa4/chismy-messenger-installer/main/version.json
```

## Flujo de publicación de una versión

1. Bump de versión en los **pubspec** de las apps
   (`chismy_messenger/pubspec.yaml`, `chismy_app/pubspec.yaml` → `version: X.Y.Z+N`)
   y en `ChismyUpdater.currentVersion` (`lib/services/chismy_updater.dart`).
2. Builds (en la repo `chismy-messenger`):
   - Windows: compilar el exe y empaquetar con Inno Setup
     (`ChismyMessenger-Setup.iss` → `ChismyMessenger-X.Y.Z-Setup.exe`;
     `Chismy-Setup.iss` → `Chismy-X.Y.Z-Setup.exe`).
   - Android: `flutter build apk` → `app-release.apk`.
3. Copiar/renombrar los artefactos a `releases/` con los **nombres fijos**:

   | Artefacto build                       | Nombre publicado       |
   |---------------------------------------|------------------------|
   | `ChismyMessenger-X.Y.Z-Setup.exe`     | `ChismyMessengerSetup.exe` |
   | `app-release.apk` (messenger)         | `ChismyMessenger.apk`      |
   | `Chismy-X.Y.Z-Setup.exe`              | `ChismyAppSetup.exe`       |
   | `app-release.apk` (chismy app)        | `ChismyApp.apk`            |

   (o usar `publish-release.ps1`, que hace la copia con rename).
4. Actualizar `version.json` (`latest` / `latestNonPreRelease`).
5. Crear la GitHub Release (tag `vX.Y.Z`) subiendo como assets los 4 archivos
   con los nombres fijos. `releases/latest/download/<asset>` apunta siempre a
   la última release:

   ```powershell
   gh release create v1.0.1 .\releases\ChismyMessengerSetup.exe `
     .\releases\ChismyMessenger.apk .\releases\ChismyAppSetup.exe `
     .\releases\ChismyApp.apk --title "v1.0.1" --notes "Cambios..."
   ```

## Cómo se actualizan las apps

Cliente: `lib/services/chismy_updater.dart` en cada app (patrón
`UpdaterService` de CompactFiles):

1. A los 5 s del arranque (y máximo cada 12 h) GET al manifiesto.
2. Compara `latest` contra `ChismyUpdater.currentVersion` (semver con
   soporte `preRelease`).
3. Si hay versión nueva y no fue omitida por el usuario, diálogo
   "Nueva versión disponible" → **Instalar ahora**:
   - **Windows**: descarga el EXE a `%TEMP%` y lo lanza con
     `/SILENT /CLOSEAPPLICATIONS /RESTARTAPPLICATIONS` (Inno Setup cierra la
     app, sobreescribe los binarios y la relanza), luego `exit(0)`.
   - **Android/iOS**: abre la URL de descarga en el navegador del sistema;
     el usuario toca el APK descargado para instalar.

## Badge web (chismy.com)

En la página de mensajes de chismy.com hay un badge tipo "Publicidad" que
detecta el SO del navegador:

- Windows → `.../releases/latest/download/ChismyMessengerSetup.exe`
- Android → `.../releases/latest/download/ChismyMessenger.apk`

## Reglas

- Los nombres de asset en `version.json` **deben** coincidir exactamente con
  los subidos a la release.
- `releases/` en la repo solo conserva los artefactos de la última versión
  (la historia de versiones vive en las GitHub Releases).
- Nunca eliminar assets de la última release mientras haya clientes
  actualizándose.
