# Nevyřešené nálezy z review `dev` proti `main` — 2026-10-04

Zbytek nálezů z `/code-review` změn `git diff main...dev`, které zatím nejsou opravené.
Opravené nálezy (ve větvi `dev-ux`) jsou na konci pro přehled.
Řádky odkazují na verzi souborů v `dev`. Závažnost: vysoká / střední / nízká.

---

## Bezpečnost

### 1. `autoload.state` se načítá přes `pickle` bez kontroly — vysoká (už v `main`)
- **Kde:** `src/tomotree/streamlit/session/state_manager.py:152-162` a `:124` (`optional_autoload`).
- **Problém:** při otevření datasetu se spustí `pickle.load` na `<dataset>/states/autoload.state`
  (případně kopii v kořeni). Kdo může do datasetu zapisovat, může nahrát upravený soubor `.state`.
  Nový dialog 📂 `list_files` (`app.py:194-206`) používá `st_filemanager` bez `allowed_extensions`
  a `.state` je povolená přípona pro typ `state`.
- **Scénář:** otevření takového datasetu spustí na serveru libovolný kód. Platí i při otevření přes
  sdílený odkaz (jiný uživatel, admin).
- **Návrh:** nahradit pickle bezpečným formátem (JSON/YAML jen s hodnotami widgetů), nebo soubory
  stavu podepisovat (HMAC klíčem serveru) a nepodepsané odmítnout; `.state` nepovolit v uploadu.

## Editor konfigurace

### 3. Neuložené změny se při jedné vrstvě ztratí bez varování — nízká až střední
- **Kde:** `src/tomotree/streamlit_app/config_editor.py:1424` (použito na ř. 1748).
- **Problém:** `_layer_is_dirty` vrací False, když je `check_layer_id` None, a konfigurace s jedinou
  vrstvou má `layer_id` vždy None (klíče `CE_nodes_text_None` apod.).
- **Scénář:** úprava uzlů nebo matice běžné jednovrstvé konfigurace a „🔚 Exit editor“ → potvrzovací
  dialog se nezobrazí a úpravy se ztratí. Kontroluje se jen oblast „other parameters“.
- **Návrh:** v `_layer_is_dirty` brát `None` jako platné id jediné vrstvy.

### 4. Úpravy se po přepnutí vrstvy ztratí, i když dialog slibuje opak — nízká až střední
- **Kde:** `src/tomotree/streamlit_app/config_editor.py:1295-1310`, `:1734-1745`.
- **Problém:** dialog `confirm_switch_layer_dialog` říká, že úpravy zůstanou v relaci, ale
  `CE_*_text_{layer_id}` jsou obyčejné klíče widgetů a Streamlit je smaže, když se widget nekreslí.
  Ze stejného důvodu cyklus přes ostatní vrstvy v `_any_unsaved_changes` nikdy nenajde změnu.
- **Návrh:** text ukládat do stínových klíčů (vzor registru widgetů: shadow key + `on_change`).

## Skeny

### 5. Sken s příponou `.OBJ` velkými písmeny spadne při načtení — střední (ověřeno spuštěním)
- **Kde:** `src/tomotree/scans/scantree.py:189` (`load_texture`).
- **Problém:** `load_scan` převádí příponu na malá písmena, takže `TREE.OBJ` přijme, ale textura se
  hledá přes `self.file.replace('.obj', '.jpg')`. Pro `TREE.OBJ` se nic nenahradí, `file_path` je
  samotný mesh a `pv.read_texture()` vyhodí `TypeError: Cannot create a pyvista.Texture from PolyData`.
  Týká se stránky Cross section a `tomogram_on_scan`.
- **Návrh:** `os.path.splitext(self.file)[0] + '.jpg'`.

### 6. Zavádějící chyba u skenů v cm nebo mm — nízká
- **Kde:** `src/tomotree/scans/scantree.py:868` (`tomogram_on_scan`), `src/tomotree/streamlit/scans/scans_trunk.py:216`.
- **Problém:** stránka senzorů podporuje skeny v cm/mm (`source_units`, `sensor_geometry_page.py:279-281`),
  ale `same_scan(..., 'm')` má jednotky napevno. Pro sken v cm se správně uloženou geometrií
  se ukáže „sensor_geometry.json belongs to another scan file“; skutečná příčina jsou jednotky
  a mesh se nepřeškáluje.
- **Návrh:** předat `source_units` z uložené geometrie a mesh přeškálovat na metry.

## Kalibrace rychlosti (už v `main`)

### 7. Kalibrace přepíše `speed_reference_value` bez ohledu na zvolenou referenci — nízká
- **Kde:** `src/tomotree/streamlit/tomograph/speed_calibration.py`.
- **Problém:** stránka kalibrace nastaví `speed_reference_value`, i když je jako reference zvolená
  „maximal velocity“ nebo „upper decile“; práh binárního tomogramu pak může vycházet z jiné hodnoty,
  než ukazuje stránka tomogramu.
- **Návrh:** referenční hodnotu počítat jen přes `analysis.reference_speed()` podle zvolené reference.

## Neověřené

- **PiCUS `.pit`:** `read_pit` čte `[BPoints]` jen pro indexy 1..n (počet senzorů). Obrys s víc body
  než senzorů se uřízne, s méně body se zahodí. V repozitáři není skutečný `.pit` soubor na ověření.
- **Arbotom:** zda `SensorCmY` neměří osu y směrem dolů.

---

## Opravené (větev `dev-ux`)

| Nález | Commit |
|---|---|
| `config.yaml` datasetu mohl přepnout otevřený strom (`selected_tree`, `rerun`, `user_uploaded_data`) | `8fce20b` |
| „interactive, rays only“ padal s diskrétní barevnou škálou | `9b61d5a` |
| Po vypnutí paprsků zůstala stará rychlost zdravého dřeva, GUI a snippet se lišily | `605b57a` |
| Barevná mapa sdílená všemi uživateli serveru | `4dbf286` |
| Mechanická analýza s polygonem ze staršího řezu; pád při smazaném polygonu | `81f7dab` |
| Křížící se polygon shodil stránku; vadná ukázka v `section_properties.main()` | `e8754ff` |
| Bez `pivot_point` se řez dělal podle osy Z místo Y | `0d318d1` |

## Opravené (větev `dev`)

| Nález | Commit |
|---|---|
| 2. Režim „jen pro čtení“ ve správci souborů: legacy správce i stránka „File browser“ odstraněny, 🗂️ otevírá `st_filemanager` v dialogu s oprávněním podle adresáře (`user_can_edit_dir`) | `d3cc52a` |
