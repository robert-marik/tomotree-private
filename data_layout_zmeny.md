# Ukládání nahraných souborů do typových podadresářů datové sady

Bylo aplikováno.

## Proč

Nahrané soubory (tomogramy, pozadí, resistograf, skeny, fotky řezu, uložené
stavy) končily buď naplocho v kořeni datové sady (`userdir/<user>/<sada>/`),
nebo jen v paměti. Jen v paměti byly skeny, fotka řezu, trasovací JSON a
rekonstruované OBJ, takže se musely nahrávat znovu. Každá stránka měla navíc
vlastní uploader s různou, nebo žádnou kontrolou kvóty a názvů souborů.

## Strategie

- **Horní úroveň zůstává volná.** Datovou sadu (typicky jeden strom) pojmenovává
  uživatel. Sdílení, kopírování, ZIP i klíčová slova dál pracují s celou sadou.
- **Uvnitř sady jsou pevné podadresáře.** Definuje je kód, aby aplikace našla
  soubory sama, bez ptaní.
- **Pravidlo čtení:** `<sada>/<podadresář>/`, pokud existuje, jinak kořen sady.
  Stará data fungují bez přesouvání.
- **Nahrávání** probíhá jednotně přes `st_filemanager` do podadresáře daného typu.

| typ (kind) | podadresář | přípony pro upload | rozpoznání starých souborů v kořeni |
|---|---|---|---|
| `tomogram` | `tomogram/` | yaml, csv, f3d, mp4 | `config.yaml`, `nodes*`/`distances*`/`speeds*`/`velocities*`/`times*`.csv, `*.f3d`, `*.mp4` |
| `background` | `backgrounds/` | png, jpg, jpeg | obrázky kromě `thumbnail.*` a kromě textur odkazovaných z MTL |
| `resistograph` | `resistograph/` | dpa | `*.dpa` |
| `scan` | `scans/` (naplocho) | obj, mtl, glb, gltf, png, jpg, jpeg | `*.obj`, `*.mtl`, `*.glb`, `*.gltf` a textury z `map_*` v MTL |
| `photo` | `photos/` | png, jpg, jpeg, json | žádné (dříve se neukládalo) |
| `state` | `states/` | state, txt | `*.state`, `*.state.desc.txt` |

`keywords.txt`, `thumbnail.*` a `favorites.json` zůstávají v kořeni. Autosave
session (`cfg.session_save_dir`) zůstává mimo sadu.

### Pravidlo zápisu (`write_dir`)

1. Pokud podadresář existuje, zapisuje se do něj.
2. Pokud neexistuje a v kořeni leží soubory daného typu, zapisuje se do kořene.
   Vytvořením podadresáře by se staré soubory schovaly.
3. Jinak se podadresář vytvoří a zapisuje se do něj.

## Nové soubory

### `src/tomotree/data_layout.py`

Modul bez závislosti na Streamlitu, aby ho mohl použít i `tomotree/resistograph.py`.

- `Kind(subdir, extensions, legacy)` a slovník `KINDS`.
- `subdir_path(dataset, kind)` vrátí cestu podadresáře, i když neexistuje.
- `data_dir(dataset, kind)` vrátí adresář pro čtení: podadresář, nebo kořen.
  Pro `None` vrátí `None`.
- `upload_dir(dataset, kind)` vrátí podadresář a v případě potřeby ho vytvoří.
- `write_dir(dataset, kind)` vrátí adresář pro zápis podle pravidla výše.
- `legacy_files(dataset, kind)` vrátí soubory daného typu ležící v kořeni.
- `move_legacy(dataset, kind)` přesune tyto soubory do podadresáře. Nic
  nepřepisuje, a pokud cíl už existuje, soubor nechá v kořeni.

### `src/tomotree/streamlit/files/data_files.py`

Jednotné ovládání souborů pro všechny stránky.

- `data_file_manager(target_dir, kind, key=None, height=420, in_dialog=False)`
  - Kořen: `write_dir()`, pokud má uživatel právo zápisu (`test_user_edit_permission()`),
    jinak `data_dir()`.
  - Parametry `st_filemanager`: `allowed_extensions` podle typu,
    `read_only=not writable`, `quota=quota_for_root(root)`, `lang` z
    `st.session_state["language"]`.
  - Pokud v kořeni leží staré soubory, zobrazí info, případně varování (když
    podadresář existuje a soubory v kořeni se ignorují), a tlačítko
    „📁 Move N file(s) to `<subdir>/`“.
  - Po úspěšné změně (upload, delete, rename, move, copy, mkdir) zahodí odvozená
    data v session (`DERIVED_STATE`, pro tomogram `config`, `data`, `tomo` a
    `tomo_method`). Mimo dialog pak spustí `st.rerun()`.
- `open_data_files_dialog(target_dir, kind)` je `@st.dialog(..., on_dismiss="rerun")`.
  Dialog zůstane otevřený pro další uploady a stránka se obnoví až po jeho zavření.
- `data_files_button(target_dir, kind, label=None, key=None, **kwargs)` je
  tlačítko, které dialog otevře.

### `tests/test_data_layout.py`

Obsahuje 12 testů:
- fallback na kořen
- existující podadresář
- `upload_dir`
- neznámý typ
- rozpoznání starých souborů pro tomogram, stavy a resistograf
- textura skenu se nepočítá mezi pozadí
- `move_legacy`, včetně případu, kdy se nepřepisuje
- `write_dir`, včetně zachování kořene se starými soubory

## Upravené soubory

### Kvóta: `src/tomotree/streamlit/storage_quota.py`

- Nový `quota_for_root(root)`. `st_filemanager` měří kvótu jen nad svým rootem,
  proto funkce vrací zbývající kvótu vlastníka plus aktuální velikost rootu.
- Vlastník se určí z cesty pod `cfg.userdir`. Mimo uživatelské úložiště (dočasná
  nebo přednahraná data) funkce vrací `None`, tedy bez limitu.

### Tomogram

- **`src/tomotree/streamlit_app/read_config.py`**
  - `read_config`: pokud dostane adresář, hledá `config.yaml` v `data_dir(path, "tomogram")`.
    Tím se řeší i `check_permission` (pravidlo `allowed`).
  - `process_config_file`: na začátku provede `target_dir = data_dir(target_dir, "tomogram")`.
    Nodes, distances, speeds, times, f3d, mp4 i automatické vytvoření
    `config.yaml` z f3d proto pracují v resolvovaném adresáři.
  - `config_update` a `add_to_config_file` zapisují do `config.yaml` v resolvovaném
    adresáři. Týká se to i uložení nastavení pozadí.
- **`src/tomotree/streamlit_app/config_editor.py`**: `config_editor` resolvuje
  adresář přes `data_dir(..., "tomogram")`. Nový `config.yaml` v sadě bez
  `tomogram/` proto vznikne v kořeni.
- **`src/tomotree/streamlit/config.py`**: `show_config` resolvuje adresář stejně.
- **`src/tomotree/streamlit_app/common.py`**: v hlášce „No tomogram available“
  nahradilo tlačítko „📁 File manager“ (skok do starého `edit_directory`) nové
  `data_files_button(..., "tomogram")`.

### Resistograf

- **`src/tomotree/resistograph.py`**: `Resistograph.__post_init__` nastaví
  `self.directory = data_dir(self.directory, "resistograph")`. Volající kód
  (`plot.py`, `curves.py`, `mean_std.py`, `drilling_path.py`) se nemusel měnit.
- **`src/tomotree/streamlit/resistograph/__init__.py`**: v hlášce „No drilling
  paths available“ je tlačítko „📁 Upload resistograph files (*.dpa)“ místo
  skoku do starého správce souborů.

### Pozadí: `src/tomotree/streamlit/bg.py`

- Odstraněn `_upload_background_image_dialog`. Měl nesanitizovaný název „Save as“
  a vlastní uploader.
- Nové helpery `background_dir(target_dir)` a `background_image_path(target_dir, name=None)`.
- Všechna místa, která skládala `os.path.join(target_dir, background_image)`,
  používají helper: `is_bg_well_defined`, `add_background`, `_select_image`,
  seznam obrázků a zobrazení vybraného obrázku.
- Pokud chybí obrázky, zobrazí se tlačítko „📤 Upload image“ (file manager).
  Vedle „Reset & select another“ přibylo „📤 Upload or manage images“, protože
  jediný obrázek se vybere automaticky.
- V `config.yaml` zůstává jen jméno souboru, takže funguje s oběma rozloženími.

### Uložené stavy: `src/tomotree/streamlit/session/state_manager.py`

- `optional_autoload`, `_load_dialog` a `_load_fragment` čtou z `data_dir(target_dir, "state")`.
- Uložení, včetně „Save as autoload.state“, zapisuje do `write_dir(target_dir, "state")`.
- Nový `_state_file_allowed(filename)`: kontrola názvu přes `_safe_component` a
  kontrola kvóty (`validate_upload_size(0)`, tedy blokuje při překročeném limitu).

### Skeny

- **`src/tomotree/streamlit/scans/scans_sensors.py`**
  - Odstraněn upload jen do paměti (`render_scan_upload`).
  - Stránka má tlačítko „📤 Upload or manage scans“ a vybírá OBJ z
    `data_dir(dataset, "scan")`.
  - Načtení OBJ s MTL a texturami zajišťuje existující `scan_files_from_directory`.
- **`src/tomotree/streamlit/scans/scan_upload.py`**: odstraněna funkce
  `render_scan_upload` a nepoužitý import `Path`. `scan_bundle_in_metres`,
  `upload_packet` a `texture_image` zůstávají.
- **`src/tomotree/streamlit/scans/scans_batch_browser.py`**: prohlíží
  `data_dir(dataset, "scan")` a má tlačítko pro upload. Konvertované GLB se
  zapisují do stejného adresáře.
- **`src/tomotree/streamlit/scans/scans_trunk.py`**: hledá `.obj` a `.glb` v
  `data_dir(dataset, "scan")`. Seznam je nově seřazený.
- **`src/tomotree/streamlit/scans/__init__.py`**: `scan_file_upload(...,
  dataset=None)` zobrazuje tlačítko pro upload, a to i v hlášce „No scan files found“.
- **`src/tomotree/streamlit/scans/sensor_geometry_page.py`**
  - Odstraněn záložní uploader OBJ pro rekonstrukci. Stránka se teď otevře jen s
    vybraným OBJ.
  - Nová funkce `_save_reconstructed_obj()` a tlačítko „💾 Save reconstructed
    OBJ to the dataset“. Soubor zapíše do `write_dir(dataset, "scan")`, kontroluje
    kvótu a právo zápisu.

### Fotka řezu: `src/tomotree/streamlit/tomograph/photo_assessment.py`

- Fotka se vybírá selectboxem z `data_dir(target_dir, "photo")`. Upload a správa
  souborů jdou přes „📤 Upload or manage photos“.
- `prepare_photo` je `@st.cache_data(max_entries=8)`. Upravený obrázek se už
  neukládá do session state, a tedy ani do autosave pickle. V session zůstává
  jen hash, jméno, kresba a nastavení.
- Nový `_restore_tracing(session, raw, force=False)` obsahuje validaci
  převzatou z dřívějšího uploaderu JSON. Formát `tomotree-photo-assessment-v1`
  se nemění.
- Trasování se ukládá tlačítkem „💾 Save photo tracing“ jako
  `<foto>.tracing.json` vedle fotky. Kontroluje se kvóta a právo zápisu.
- Při výběru fotky se uložené trasování načte automaticky. Tlačítko „Reopen
  saved tracing“ ho načte znovu (vynutí nový klíč editoru). Download JSON zůstal.

### Globální file manager: `src/tomotree/streamlit_app/app.py`

- `list_files`: dialog má `on_dismiss="rerun"`, `quota=quota_for_root(...)`,
  `read_only=not test_user_edit_permission()` a jazyk podle aplikace. Dříve šlo
  zapisovat i do přednahraných a sdílených dat.
- Smazán nepoužívaný dialog `upload_file` i zakomentované tlačítko 📤.

### Doprovodné opravy

- `src/tomotree/streamlit/save.py`: název nové sady se kontroluje přes
  `_safe_component`. Dříve prošly `/` a `..`.
- `src/tomotree/streamlit/share.py`: totéž pro název sdíleného odkazu.
- `src/tomotree/streamlit_app/admin.py`: `logger.debung` opraveno na `logger.debug`.

## Odchylky od původního plánu

- Modul je `tomotree/data_layout.py`, ne `tomotree/streamlit/data_layout.py`,
  aby nezávisel na Streamlitu.
- Skeny jsou ve `scans/` naplocho, bez `scans/<jméno>/`. Batch browser i trunk
  scanner procházejí jen jednu úroveň a MTL a textury se párují ve stejném adresáři.
- Ochrana proti zip-slip nepřidána. `zipfile.extractall` v Pythonu 3 sám
  odstraňuje absolutní cesty a `..`.
- Kvóta se nepočítá u dočasně nahraných dat (`/tmp`) ani u přednahraných dat,
  kde je file manager jen pro čtení.

## Mimo rozsah (možná další fáze)

- Starý vlastní správce souborů `files/file_manager.py` a `files.file_upload`.
  Hlášku „zip extracted“ vypisuje, ale nic nerozbalí.
- `ini_data.handle_files_upload` (dočasná data přes zip).
- Jednorázový migrační skript pro existující sady. Zatím stačí tlačítko pro
  přesun v každém dialogu.
- Chyby nalezené při průzkumu, zatím neopravené:
  - `Config.__post_init__` v `app_config.py` je odsazený na úrovni modulu, takže
    se nikdy nespustí.
  - `admin.py:122` spadne, pokud je limit zadaný jako řetězec `"100*1024"`.
  - `sidebar.render_quick_access` předává `username={...}`, tedy množinu.

## Ověření

- `tests/test_data_layout.py`: 12 testů prošlo.
- Smoke test přes `streamlit.testing.AppTest` na dočasné sadě se starým `p1.dpa`
  a `config.yaml` v kořeni:
  - zobrazí se nabídka přesunu a přesun proběhne do `resistograph/`
  - pro typ `photo` vznikne `photos/`
  - `quota_for_root` vrací limit pro uživatelskou sadu a `None` pro `/tmp`
- Celá sada testů (bez e2e a local_integration): 322 prošlo, 10 selhalo v
  `tests/streamlit/test_premium.py`. Stejně selhávají i na čistém HEAD, se
  změnou nesouvisí.
- Import všech upravených modulů je v pořádku. `scans_trunk` se mimo
  `streamlit run` neimportuje kvůli `stpyvista`, platí to i bez změn.

### Zbývá ručně otestovat v aplikaci

1. Stará sada `marik/linden_arbotom` (`config.yaml`, `nodes.csv`, `times.csv` a
   `autoload.state` v kořeni):
   - načte se beze změny
   - po „Přesunout“ se načte stejně z `tomogram/` a `states/`
2. Nová sada:
   - nahrání `.dpa` vytvoří `resistograph/` a graf se zobrazí bez dalšího nahrávání
   - totéž platí pro pozadí, skeny (OBJ, MTL a textura) a fotku řezu
   - uložené trasování se po reloadu obnoví
3. Přednahraná a sdílená data: file manager je jen pro čtení.
4. Uživatel `test_limited`: upload nad limit se odmítne.

## Poznámka k diffu

Soubory `scans/*`, `streamlit_app/app.py`, `streamlit_app/page_handlers.py`,
`tomograph/photo_assessment.py` a `menu.py` obsahovaly necommitnuté změny ještě
před touto úpravou. Moje změny jsou nad nimi, proto diff před commitem zkontroluj.
