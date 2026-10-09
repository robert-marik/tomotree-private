# TomoTree – co chybí ke komerčnímu úspěchu u arboristů

## Kontext

TomoTree už teď umí hodně: má rekonstrukci (EBSI, ERSM), binární tomogram, posouzení podle fotografie, 3D skeny, mechaniku průřezu a rezistograf. Načte data z PiCUS, Arbotom a ArborSonic 3D, je přeložená do 8 jazyků, má role, kvóty, sdílení a Docker.
Je to ale nástroj **pro výzkumníka**, ne pro arboristu v praxi. Arborista potřebuje během 10 minut od měření mít **posudek pro klienta** a doporučení (ponechat, ošetřit, kácet). Průzkum kódu ukázal tyto mezery:

- není žádný výstupní report (PDF/DOCX), jen PNG, CSV a ZIP,
- není výsledné číslo „o kolik je strom oslabený“ (ztráta pevnosti, poměr t/R, bezpečnost),
- chybí pojem projekt/lokalita/strom s GPS, mapou a historií měření,
- data se dostanou dovnitř jen uploadem přes prohlížeč (limit 10 MB/soubor) nebo přes serverové složky. Nic se nepřipojuje ke cloudu (`ini_data.py:get_directory`, žádné requests/S3/WebDAV),
- nabídka je členěná podle metod (Rays, EBSI, ERSM, Angle plots…), ne podle pracovního postupu,
- chybí obchodní vrstva: organizace/týmy, placení, zkušební verze, podmínky/GDPR, onboarding.

Cíl: odstranit překážky, kvůli kterým arborista aplikaci nepoužije, a přidat funkce, které nenabízí software výrobců přístrojů.

---

## 1. Import dat z cloudu – specifikace

### Uživatelské scénáře
1. **Odkaz:** arborista vloží sdílený odkaz z Google Drive, OneDrive/SharePointu, Dropboxu, Nextcloudu/ownCloudu (CESNET) nebo WeTransferu. Aplikace soubor nebo složku stáhne a otevře jako dataset. Nepotřebuje OAuth a pokryje 80 % případů.
2. **Připojený účet:** uživatel jednou připojí účet (OAuth: Google, Microsoft, Dropbox; WebDAV: Nextcloud) a vybírá ve stromu složek, jak je zvyklý z průzkumníka.
3. **Inbox / hlídaná složka:** terénní pracovník nahraje z mobilu nebo notebooku přístroje do cloudové složky `TomoTree/Inbox/`. Aplikace ji periodicky projde, z každé podsložky (nebo ZIP) vytvoří dataset v `streamlit_user/<user>/` a pošle upozornění „3 nová měření připravena“.
4. **Export zpět:** report PDF, ZIP datasetu a CSV výsledky jde jedním tlačítkem uložit do stejné cloudové složky.

### Chování
- Rozpoznávají se tytéž formáty jako u uploadu: `.pit`, `.abt`, `.f3d`, `config.yaml`, CSV, `.dpa`, fotky, OBJ/GLB a ZIP. Použije se stávající logika z `handle_files_upload()` a `config_from_device_file()`.
- Složka s více měřeními (např. export celé ArborSonic databáze) se nabídne jako **dávkový import**: tabulka stromů se zaškrtávátky a náhledem metadat (druh, GPS, datum) z hlaviček souborů (`device_files.py`).
- Stahuje se do dočasné složky, pak proběhne stávající „Save data“ s kontrolou kvóty (`save.py:save_data_to_user_directory`, `storage_quota.py`).
- Limity: velikost stahování (konfigurovatelná, např. 500 MB), `check_zip_limits` (`streamlit/files/zip_extractor.py`), časový limit a zobrazení průběhu.
- Bezpečnost: ochrana proti SSRF (blokovat privátní IP, povolit jen https), whitelist domén poskytovatelů, žádné následování přesměrování mimo whitelist. OAuth tokeny ukládat šifrované a mimo session soubory (klíče začínající `_`).

### Architektura (v souladu s CLAUDE.md)
- **Knihovna** `src/tomotree/cloud/` (bez streamlitu):
  - `links.py` převádí sdílený odkaz na přímou URL ke stažení pro každého poskytovatele,
  - `fetch.py` dělá streamované stahování s limitem velikosti a kontrolou SSRF (`httpx` už je v `requirements-mamba.txt`),
  - `providers/` má společné rozhraní `list(path)`, `download(path, dest)`, `upload(src, path)` s implementacemi WebDAV (Nextcloud), Google Drive, MS Graph a Dropbox,
  - `inbox.py` projde inbox a vrátí seznam nových datasetů (idempotentně podle hashe nebo ETag).
- **GUI:** nová položka „Import z cloudu“ v nabídce zdrojů `ini_data.py:get_directory()`, která končí voláním `select_button(path)` jako ostatní zdroje. Výběr poskytovatele bude radio přes registr widgetů. Pole pro URL je jednorázové.
- **Konfigurace:** nová pole v `Config` (`streamlit_app/app_config.py`) `cloud_import_enabled`, `cloud_max_download_mb` a `cloud_providers`, plus šablona v `config/config.yaml.dist`. Secrets půjdou do sekcí `[cloud.google]`, `[cloud.microsoft]` a `[cloud.dropbox]` v `.streamlit/secrets.toml.dist`. Ke každému přibude kontrola v `setup_check.py`.
- **Oprávnění:** přes `access_policy.py`. Například odkaz smí role `user` a výš, inbox jen `premium`.
- **Etapy:** (a) import z odkazu a WebDAV, (b) OAuth výběr složek a export zpět, (c) inbox s upozorněním.

---

## 2. Co dalšího chybí (podle priority)

### A. Bez toho to arborista nepoužije
1. **Report / posudek (PDF + DOCX)**
   - Obsahuje šablonu s logem firmy, identifikaci stromu (druh, GPS, obvod, výška měření, datum, měřil), foto, tomogram s legendou, binární tomogram, rezistograf a klíčová čísla.
   - Má volný text závěru a doporučení.
   - Výstup odpovídá struktuře běžných posudků (v ČR standard AOPK **SPPK A02 002 Hodnocení stavu stromů**, mezinárodně ISA TRAQ).
   - Logika patří do knihovny `tomotree/report.py`, GUI sbírá jen volby.
2. **Průvodce „Nové měření“ (wizard)**
   - Postup: nahrát nebo importovat → kontrola dat (už existuje quality check) → automatický tomogram s doporučeným nastavením → binarizace → výsledná čísla → report.
   - Stávající stránky podle metod zůstanou jako „Expertní režim“.
3. **Interpretace, nejen obrázek**
   - Zbytková stěna **t/R** s prahem Mattheck 0,3 a **ztráta pevnosti v ohybu (%)** z modulu průřezu oproti zdravému průřezu. Data už počítá `physics/section_properties.py`, chybí jen srovnání s plným průřezem a srozumitelný výrok.
   - Volitelně **zjednodušená statická analýza**: zatížení větrem podle výšky a koruny, bezpečnost, semafor zelená/oranžová/červená.
4. **Onboarding:** ukázkový strom otevřený jedním klikem (Jičín_05 už existuje), 2minutové video a kontextová nápověda „co mám dělat dál“ na každé stránce.

### B. Čím se odlišit od softwaru výrobců
5. **Nezávislost na výrobci:** jedno prostředí pro PiCUS, Arbotom, ArborSonic i rezistografy.
   - Doplnit formáty: IML-RESI PD (`.rgp`), Rinntech Resistograph (ověřit `.dpa`), ArborSonic 2D/3D z novějších verzí a PiCUS TreeTronic.
   - Tohle je hlavní obchodní argument („jeden nástroj pro všechny přístroje ve firmě“).
6. **Projekty, inventarizace a mapa**
   - Hierarchie projekt (zakázka, lokalita) → strom → měření.
   - Atributy: druh (číselník s latinou), GPS z hlavičky PiCUS nebo z fotky (EXIF), mapa stromů projektu barevně podle výsledku.
   - Import a export CSV/GeoJSON/Shapefile pro GIS a pasporty zeleně obcí.
7. **Monitoring v čase:** opakované měření téhož stromu, porovnání tomogramů vedle sebe, trend t/R a ztráty pevnosti. Pro obce je to opakovaný důvod platit.
8. **Dávkové zpracování:** desítky stromů najednou (navazuje na cloud inbox) se souhrnnou tabulkou a hromadným reportem.

### C. Obchodní a provozní vrstva
9. **Organizace/týmy:** firemní účet, sdílené projekty uvnitř firmy, role v rámci organizace. Dnes jsou jen jednotliví uživatelé a symlinkové sdílení.
10. **Tarify a platby:**
    - Free: 3 stromy/měsíc, bez reportu s logem.
    - Pro: neomezeně, report a cloud.
    - Team: organizace, inbox, API.
    - Platby přes Stripe. Role `premium` a kvóty už existují.
11. **Důvěra:**
    - Stránka s validací metod (srovnání s řezy, publikace) a citovatelná metodika (`docs/methodology` už existuje).
    - Podmínky, GDPR, zálohy, export všech dat (bez vendor lock-in) a hosting v EU.
12. **AI asistent textu posudku:** návrh závěru z naměřených čísel, který arborista edituje. Napojení na LLM už existuje (Groq v `config_editor.py`). Musí být jasně označeno jako návrh.
13. **Terénní režim:** mobilní stránka pro zápis stromu (foto, GPS z telefonu, obvod, rozmístění senzorů, poznámky), která se synchronizuje do projektu.

---

## Doporučené pořadí
1. Import z cloudu přes odkaz a WebDAV (malý rozsah, okamžitý přínos)
2. Výsledná čísla t/R a ztráta pevnosti (knihovna)
3. Průvodce
4. PDF/DOCX report
5. Projekty a mapa, monitoring
6. OAuth cloud a inbox
7. Organizace a platby

Body 1–4 dohromady tvoří minimální komerční produkt: „nahraj → posudek za 10 minut“.

## Kritické soubory pro první etapu (cloud podle odkazu)
- nové: `src/tomotree/cloud/{__init__,links,fetch}.py` a `tests/test_cloud_links.py`, `tests/test_cloud_fetch.py` (mock HTTP, SSRF, limity)
- `src/tomotree/streamlit/ini_data.py` (nová položka v `get_directory`, znovu použít `handle_files_upload` a `select_button`)
- `src/tomotree/streamlit_app/app_config.py`, `config/config.yaml.dist`, `streamlit_app/setup_check.py`
- `src/tomotree/streamlit/access_policy.py`
- řetězce přes `_()`, pak `make pot && make po`, překlady a `make json`

## Ověření
- unit testy převodu odkazů pro každého poskytovatele, testy limitů a odmítnutí privátních IP
- AppTest v `tests/streamlit/`: import ZIPu z mockované URL → dataset se otevře a jde spočítat tomogram
- smoke test všech stránek (`tests/streamlit/test_all_pages_smoke.py`) a `make pytest`
- ručně: veřejný odkaz na Jičín_05 na Google Drive a v Nextcloudu → otevřít → EBSI tomogram
