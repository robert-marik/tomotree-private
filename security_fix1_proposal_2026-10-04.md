# Bod 1: spuštění kódu přes uložené stavy (`*.state`, pickle)

## Kontext
`streamlit/session/state_manager.py` ukládá stav aplikace do `<dataset>/state/*.state` jako pickle
(vnější `dict` klíč → `pickle.dumps(hodnota)`) a načítá ho přes `pickle.load` / `pickle.loads`:
- automaticky `autoload.state` při otevření datasetu (`optional_autoload`, `app.py`),
- ručně tlačítkem 📥 v dialogu „Read and manage saved states“ (každý `*.state`).

Kdokoli, kdo dostane `.state` do datasetu (upload dočasných dat, ZIP, správce souborů ve vlastních
datech, sdílení), spustí při otevření libovolný kód na serveru. Oprava nesmí uživateli nic změnit:
staré soubory se musí dál načítat se stejným výsledkem.

Co v souborech reálně je (33 souborů v `~/tomotree_data`, `~/tomotree_radon`, analýza opkódů bez spuštění):
hodnoty widgetů (str/čísla/listy), numpy skaláry a pole, pandas `DataFrame`/`RangeIndex`,
`tomotree.classes` (`Tomo`, `Node`, `Ray`, `Cell`, `Grid`, `Segment`), shapely (`from_wkb`),
`tomotree.resistograph.Resistograph`/`DrillingPath`, `scipy OptimizeResult`, staré `Config`,
streamlit `PlotlyState`. Tedy ne jen nastavení – i výpočty (`tomo`, `data`).

## Varianty

### A) Podepsaný pickle (HMAC serverovým klíčem)
Formát: `b"TTSTATE1"` + HMAC-SHA256(payload) + payload (stejný pickle jako dnes). Načtení jen když
podpis sedí. Klíč: `state_signing_key` v `config.yaml`, jinak se vygeneruje `config/state_signing.key` (0600).
Staré soubory podepíše jednorázový admin skript.
- (+) Plná věrnost, nulová změna pro uživatele, malý kód, žádný seznam povolených tříd.
- (−) Pořád pickle: únik klíče (čtení `config/`) = spuštění kódu. Rotace klíče = přepodepsat vše.
- (−) Soubor přenesený jinam (stažený ZIP datasetu nahraný na jiný server, dev ↔ produkce, ruční kopie)
  nemá platný podpis → bez dalšího mechanismu se nenačte (změna pro uživatele).
- Riziko: migrace nesmí podepsat už podstrčený škodlivý soubor → každý starý soubor nejdřív ověřit (viz B).

### B) Omezený unpickler (seznam povolených tříd)
`pickle.Unpickler` s `find_class`, který pustí jen třídy/funkce ze seznamu (numpy reconstruct a dtype,
pandas DataFrame/Index/BlockManager, `tomotree.classes.*`, `Resistograph`, `DrillingPath`, shapely `from_wkb`,
`OptimizeResult`, `PlotlyState`, bezpečné builtins). Cokoli jiného → klíč se přeskočí s varováním.
- (+) Zavře spuštění kódu pro všechny soubory včetně přenesených, bez klíče, bez migrace, stejný výsledek.
- (−) Seznam se musí udržovat: přejmenovaná/nová třída v uloženém stavu → klíč se nenačte (fail closed,
  hláška „Could not restore …“), ne pád.
- (−) Zbytkové riziko: povolené třídy dostanou data útočníka (`__setstate__` = `__dict__.update`) →
  podivné objekty, velká pole (DoS paměti), ne spuštění kódu. numpy/pandas reconstruct funkce jsou
  standardně považované za bezpečné, ale je to důvěra v cizí kód.

### C) Nový formát (ZIP: JSON pro jednoduché hodnoty + `.npy`/`.parquet` pro pole a tabulky)
`Tomo`, `Resistograph`, shapely, plotly stav by se neukládaly a po načtení přepočítaly z datasetu.
- (+) Spuštění kódu principiálně vyloučeno, čitelné, přenositelné, nezávislé na verzi knihoven.
- (−) Velká práce (serializace pro každý typ), konverze starých souborů je ztrátová (`tomo`, `data`).
- (−) Porušuje „nic se nezmění“: načtení stavu s `tomo` bude pomalejší (přepočet) a může dát jiný výsledek,
  pokud uložený `tomo` vznikl z dat/nastavení, která v souboru nejsou (např. jiná verze knihovny,
  ručně upravená data v session). Vyšší riziko regresí.

## Doporučení: B + A (omezené čtení vždy, podpis pro soubory z tohoto serveru)
1. **Načítání** (`_load_state`):
   - soubor s platným podpisem tohoto serveru → normální `pickle` (plná věrnost, žádný seznam tříd);
   - bez podpisu / s cizím podpisem (staré soubory, přenesené, nahrané) → **jen omezený unpickler (B)**.
   Spuštění kódu je tak zavřené pro všechno, co nepodepsal server; legitimní staré i přenesené soubory
   se načtou stejně jako dnes (s výjimkou případné třídy mimo seznam – varování u klíče).
2. **Ukládání** (`_save_state`): vždy podepsaný formát `TTSTATE1`.
3. **Převod starých souborů**: admin skript `python -m tomotree.streamlit.session.state_manager --sign <dirs>`
   projde `dirname`, `userdir`, `shareddir`, každý nepodepsaný `*.state` načte omezeným unpicklerem
   (ověření, že je neškodný), přepíše na podepsaný se stejným obsahem, zachová `mtime` (řazení v dialogu)
   a `.desc.txt`; soubory, které omezenou kontrolou neprojdou, jen vypíše (nepodepíše). Volitelně i
   automaticky: nepodepsaný soubor úspěšně načtený omezeně se při načtení nepřepisuje (data uživatele
   se bez jeho akce nemění) – podepíše se až při dalším uložení.
4. Klíč: `state_signing_key` v `config.yaml` (dokumentovat v `docs/config_file*.md`); když chybí,
   vygenerovat `secrets.token_bytes(32)` do `config/state_signing.key` s právy 0600 a logovat to.
   Bez klíče (nejde zapsat) → podpis vypnut, vše se čte omezeně (bezpečné, jen bez zrychlení).

Proč ne jen A: přenesené soubory by bez B přestaly fungovat. Proč ne jen B: podpis drží věrnost
souborů tohoto serveru i po přidání nové třídy do session, bez údržby seznamu.

Stejný mechanismus (podpis + omezené čtení) použít i pro session soubory `session_auto.py`
(`_load_pickle_file`) – jsou jen na serveru, ale je to tentýž pickle (obrana do hloubky). Samostatný commit.

## Soubory
- `src/tomotree/streamlit/session/state_manager.py` – `_save_state`, `_load_state`, CLI `--sign`.
- nový čistý modul `src/tomotree/safe_pickle.py` (bez streamlitu): `SafeUnpickler` + `ALLOWED`,
  `dumps_signed(obj, key)`, `loads_state(data, key)` → (`dict`, `signed: bool`); testovatelný bez GUI.
- `src/tomotree/streamlit_app/app_config.py` – `state_signing_key`; `read_config.py` načtení/generování klíče.
- `src/tomotree/streamlit_app/session_auto.py` – `_load_pickle_file`/`save_user_data` (samostatný commit).
- `docs/config_file*.md`, `docs/admin_guide.md` – klíč a migrační příkaz.

## Ověření
- Seznam povolených tříd sestavit z reálných souborů: projít všech 33 `*.state` loggujícím `find_class`
  (bez spuštění), plus `tests/data`.
- Test zpětné kompatibility: každý existující `*.state` → starý `pickle.load` vs. nový `loads_state` →
  stejné klíče a hodnoty (`==`, `np.array_equal`, `DataFrame.equals`, u `Tomo` porovnat `__dict__` klíčové
  atributy / `tomo.speeds`).
- Testy bezpečnosti: pickle s `os.system` / `subprocess` / `builtins.eval` → klíč odmítnut, nic se nespustí
  (sentinel soubor nevznikne); podepsaný soubor s pozměněným bajtem → bere se jako nepodepsaný.
- Migrační skript na kopii `~/tomotree_data` → všechny soubory podepsané, mtime zachované, načtení v GUI
  (`/verify`: otevřít dataset s `autoload.state`, ručně načíst 2–3 stavy) dává stejné tomogramy.
- Celá sada `pytest` (známé pády `test_premium`, `test_binary_page`).

## Otevřené otázky pro uživatele
- Varianta: doporučená B + A, nebo jen B (nejmenší změna), nebo C?
- Klíč: generovat automaticky do `config/`, nebo ho musí admin zadat do `config.yaml`?
- Migrace: spustit skript jednorázově adminem (doporučeno), nebo podepisovat automaticky při načtení?
