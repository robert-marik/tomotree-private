# Bezpečnostní review `src/tomotree/streamlit_app/` (2026-10-04)

Read-only review (agent `security`), větev `dev-refactoring`, commit 0f8f4a6. Nic nebylo změněno.
Body 1–3 ověřeny v kódu ručně, ostatní podle zprávy agenta.

Tajné soubory: v gitu je jen `config/login.yaml.dist`; `.streamlit/secrets.toml` ani `config/login.yaml`
v gitu nejsou. **Trackovaný je `docker/credentials.toml`** (43 B, jeden řádek s `=`), zkontrolovat.

## Potvrzené nálezy

### 1. KRITICKÉ – spuštění kódu přes `autoload.state` (pickle)
- `src/tomotree/streamlit/session/state_manager.py:123,132` (`pickle.load` / `pickle.loads`), volá
  `optional_autoload()` (`state_manager.py:156`) z `app.py:318` pro každý otevřený dataset.
- Uživatel nahraje ZIP/data s `autoload.state` (kořen nebo `state/`); při otevření se spustí libovolný kód
  na serveru. Stačí upload do tmpdir (`ini_data.py:459,486`), bez práva zápisu.
- Oprava: zrušit pickle → JSON/YAML s whitelistem klíčů a typů. Přechodně aspoň HMAC podpis serverovým
  tajemstvím a odmítnout nepodepsané soubory.

### 2. KRITICKÉ – path traversal session souborů přes `?session=` (pickle)
- `app_routing.py:50-51` vrací `session` z URL bez validace → `session_auto.py:472`
  (`os.path.join(CACHE_DIR, f"session_{username}_{provider}_{session_id}.pkl")`) → `finish.py:81` →
  `save_user_data` (`session_auto.py:449-456`) a `load_user_data` → `pickle.load` (`session_auto.py:68`).
- `session=/../../x` zapíše `.pkl` mimo `session_save_dir` (mkdir s `parents=True` vytvoří mezikrok);
  při dalším běhu se libovolný `*.pkl` načte. Ve spojení s nahraným `.pkl` → spuštění kódu. Se znalostí
  cizího ID lze načíst/přepsat cizí session (`username`, `roles`, `target_directory`).
- Oprava: validovat ID regexem `^[a-z0-9]{8,32}$`, jinak vygenerovat nové; generovat `secrets.token_urlsafe`
  (teď `random.choices`, `app_routing.py:55`); ověřit `realpath` uvnitř `session_save_dir`; nahradit pickle.

### 3. VYSOKÉ – admin konzole bez serverové kontroly role, běží před přihlášením
- `app_routing.py:89-92` a `admin.py:510` (`console()`) kontrolují jen `_admin_console` v session;
  volá se v `app.py:258` před `setup_authenticator` / `check_user_access`.
- Kdo dostane `_admin_console=True` do session (bod 4, obnova session bod 8), získá credentials report
  (hashe hesel, cookie key, api_key – `admin.py:545-552`), editaci `config.yaml` včetně
  `extra_editable_files` (zápis kamkoli), file manager nad `userdir` a `dirname`. Přežije logout,
  blacklist i odebrání role.
- Oprava: `is_admin_or_developer()` + `authentication_status` v `handle_admin_console` i na začátku
  `console()`; volání přesunout za autentizaci; při logoutu mazat `_admin_console`.

### 4. VYSOKÉ – `config.yaml` datasetu přepisuje libovolné klíče session state
- `read_config.py:162-164` (blok `config:` nastaví libovolný klíč), `read_config.py:219-222`
  (`setup_raw_keys` přepíše každý existující klíč session, který je i v YAML).
- Dataset s `roles: [admin]`, `permission: true`, `_admin_console: true`, `next_action`/`next_target`
  → eskalace na admina (u hesla role přežije). Sdílený dataset napadne každého, kdo ho otevře.
- Oprava: odstranit legacy blok `config:`; `setup_raw_keys` omezit whitelistem (registr widgetů, známé
  datové klíče); nikdy auth/oprávnění ani klíče s `_`.

### 5. VYSOKÉ – editor konfigurace zapisuje bez kontroly oprávnění
- `pending_actions.py:198-200` volá `config_editor()` bez kontroly; zápisy `config_editor.py:1134-1135`
  (raw editor), `:1316`, `:1516`, helpery `:443`, `:483`, `:944`, `:1015`, `:1036`. Kontrolu má jen
  tlačítko `:1809`. Vstup bez kontroly: `common.py:146` (`is_tomogram_available(edit_buttons=True)`,
  z `tomogram.py:446`).
- Uživatel jen pro čtení otevře přednahraný/sdílený dataset bez tomogramu → „Configuration editor“ →
  přepíše `config.yaml` všem (s bodem 4 převzetí jejich session).
- Oprava: `test_user_edit_permission()` v `_run_edit_config` a v `config_editor()`, jinak jen čtení;
  kontrola u každého zápisu.

### 6. VYSOKÉ – vlastnictví dat podle shody jména adresáře v cestě
- `streamlit/__init__.py:94-97`: `username in selected_tree.split("/")[:-1]`; regex jmen
  `auth_patch.py:160` připouští `home`, `tmp`, `user_data`, `streamlit_data`.
- Registrace jako `user_data` (nebo jiný adresář v absolutní cestě) → zápis do dat všech uživatelů
  i přednahraných dat, včetně file manageru (`app.py:199-203`).
- Oprava: `os.path.commonpath([realpath(selected_tree), realpath(userdir/username)])`; zakázat
  rezervovaná jména při registraci.

### 7. STŘEDNÍ – starý `file_manager` přes `next_action='edit_directory'` vždy zapisovatelný
- `pending_actions.py:194-196` volá `file_manager(next_target)` s `readonly=False` bez kontroly.
  Vstupy: `read_config.py:127` → `streamlit/__init__.py:137-140` (tlačítko „Directory manager“ při
  chybě configu), `directory_manager.py:312,395`.
- Oprava: `readonly=not test_user_edit_permission()`, `next_target` ověřit proti povoleným kořenům.

### 8. STŘEDNÍ – obnova session nefiltruje auth klíče ani `_`
- `session_auto.py:480-484` neodfiltruje `roles`, `username`, `authentication_status`, `permission`,
  klíče s `_` (při ukládání se `_` vyřazuje, `:436`). `configuration` (cfg vč. `passphrase`) se ukládá
  do pickle (`:435-443`).
- Oprava: stejný seznam jako `state_manager.EXCLUDE_KEYS` + `permission`, `configuration`, prefix `_`;
  auth klíče po obnově vždy znovu z autentizace.

### 9. STŘEDNÍ – „Create .zip“ bez kontroly oprávnění a kvóty
- `app.py:205-209`: `shutil.make_archive(target_dir, 'zip', target_dir)` zapíše `<dataset>.zip`
  do nadřazeného adresáře (`shareddir`, `dirname`, `userdir`) i uživateli jen pro čtení; opakováním
  zaplní disk.
- Oprava: archiv v paměti nebo `tempfile`, smazat po stažení, omezit velikost.

### 10. STŘEDNÍ – rozbalení ZIPu bez limitu (zip bomb)
- `ini_data.py:459`, `:486` (`extractall`). Zip slip ošetřuje `zipfile`, velikost a počet členů ne.
- Oprava: sečíst `ZipInfo.file_size`, limit počtu souborů a kompresního poměru, kontrola kvóty.

### 11. NÍZKÉ – `?health` bez přihlášení prozrazuje informace
- `app_routing.py:70-73`, `health.py:157-204`: verze Pythonu/OS/knihoven, existence `login.yaml`/
  `secrets.toml`, registrace, git commit.
- Oprava: ven jen „OK“, detaily adminovi.

### 12. NÍZKÉ – nasazení
- `docker/config.toml:42` `enableCORS = false` → zapnout / omezit nginxem.
- `docker/config.toml:51` `gatherUsageStats = true` → vypnout.
- Explicitně nastavit `maxUploadSize` a `enableXsrfProtection`.

### 13. NÍZKÉ – osobní údaje a chyby v logu a UI
- `authenticator.py:60,63,490,500,511` logují e-maily (GDPR) → hash/ID.
- `login.py:80,95,115`, `session_auto.py:70,446` ukazují/logují výjimky a cesty serveru.
- „Config report“ v konzoli (`admin.py:535-540`) vypisuje `passphrase`.

## Podezření (neověřeno)
- `disabled=` u tlačítek vynucuje oprávnění jen v UI (`sidebar.py:171`, `state_manager.py:206,241`,
  `config_editor.py:1809`, …); upravená websocket zpráva akci provede. Kontrolu dát do obsluhy akce.
- Obejití premium: `menu.py:287` vrací `last_sub[first_level]` bez ověření proti `menu_data`;
  `render_premium_page` (`premium.py:150`, `app.py:364`) nekontroluje roli → `check_access(["premium"])`
  v `render_premium_page` a validace `selected`.
- Glob znaky v OIDC e-mailu: `session_auto.py:393,417` dosazuje `username` do `glob` → `glob.escape`
  nebo hash jména souboru.
- Název nahraného souboru v cestě: `ini_data.py:451,458,469` (`uploaded_file.name` v `os.path.join`)
  → `os.path.basename()` a validace.
- Mimo bezpečnost: `?share=` (`ui.py:57-58`) volá `realpath`, které rozbalí symlink z `share.py:51`
  do `userdir`, takže `_safe_resolve(…, [shareddir])` ho nejspíš vždy odmítne. Odkaz `?key=` nelze
  odvolat a nemá expiraci.

## Doporučené pořadí oprav
1, 2 (zrušit pickle, validovat session ID) → 3, 4 (role v konzoli, whitelist klíčů z `config.yaml`)
→ 5, 6, 7 (serverové kontroly oprávnění).
