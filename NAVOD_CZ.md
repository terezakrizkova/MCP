# MCP server pro LN tým: co, kde a jak (návod pro Terezu)

Cíl: do prosincového workshopu v MK mít funkční MCP server jen pro čtení, který si každý vývojář spustí lokálně ve VS Code.

---

## KROK 1: Testovací tenant a servisní účet (ty jako admin, cca 1 h)

**Kde:** Infor OS Portal **testovacího** tenantu (ne produkce zákazníka).

1. **Servisní uživatel jen pro čtení**
   - Infor OS Portal → User Management: založ uživatele, např. `svc_mcp_read`.
   - Role: jen to, co je potřeba na čtení LN, Data Fabric (Compass) a případně Factory Track. Žádné zapisovací role.
   - Pro Compass potřebuje uživatel roli Data Fabric s právem dotazovat Data Lake. Přesný název role ověř v dokumentaci Data Fabric pro svůj tenant.

2. **Authorized App typu Backend Service**
   - Infor OS Portal → **API Gateway → Authorized Apps → +**
   - Typ: **Backend Service**, název `KM_LN_MCP_READ`
   - Ulož → **Download Credentials** → zaškrtni **Create Service Account** a vyber uživatele `svc_mcp_read` → stáhni.
   - Výsledek je soubor `KM_LN_MCP_READ.ionapi`. Obsahuje heslo, s tím podle toho zacházej.

3. **LN REST API**
   - V LN aktivuj REST služby, které chceš číst (např. `tdapi.slsSalesOrder`, `tdapi.purPurchaseOrder`), pro testovací firmu.
   - Když máš zapnuté API autorizace, dej API identitě servisního účtu roli **jen s GET routami**.
   - Postup je v *Infor LN REST API Administration Guide (Cloud)*, kapitoly „Activating services“ a „API authorizations“.
   - Rychlý test: API Gateway → Available APIs → Infor LN → LN/lnapi → služba → **Try it out**, s hlavičkou `X-Infor-LnCompany` = číslo firmy.

4. **Data Fabric / Compass**
   - Ověř, že v Data Lake jsou LN tabulky, které chceš (tdsls400, tcibd001 …).
   - Compass v2 je v API suite **Infor Data Fabric** (stará cesta přes ION API suite je deprecated).

5. **Factory Track:** úkol pro **Nikolaje**. Má zjistit cestu k Mongoose IDO API v API Gatewayi a Mongoose config a doplnit je do `.env` (`FT_BASE_PATH`, `FT_MONGOOSE_CONFIG`) a do `knowledge/factory-track.md`.

---

## KROK 2: GitHub repozitář (ty nebo Tim, 15 min)

**Kde:** GitHub organizace **kmits-bs** (pozvánka od Boda Radkeho).

1. Založ **privátní** repo `km-ln-mcp`.
2. Nahraj do něj obsah tohoto balíčku:
   ```bash
   cd km-ln-mcp
   git init && git add . && git commit -m "Initial: MCP server read-only"
   git branch -M main
   git remote add origin https://github.com/kmits-bs/km-ln-mcp.git
   git push -u origin main
   ```
3. Před pushnutím zkontroluj, že `git status` **neukazuje** `secrets/` ani `.env`. Hlídá to `.gitignore`.
4. Settings → Branches → ochrana `main`: změny jen přes Pull Request s 1 schválením (ty nebo Štefan).

---

## KROK 3: Zprovoznění u tebe (30 min)

**Kde:** tvůj notebook.

1. Nainstaluj **uv**. Je to nástroj, který si sám stáhne Python:
   - Windows PowerShell: `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`
2. V repu vytvoř složku `secrets` a vlož do ní `.ionapi` přejmenovaný na `ln.ionapi`.
3. Zkopíruj `.env.example` → `.env` a nastav `LN_COMPANY` (číslo testovací firmy).
4. V terminálu:
   ```bash
   uv sync
   uv run km-ln-mcp      # nic nevypíše a čeká, to je v pořádku → Ctrl+C
   ```
5. Otevři složku ve **VS Code** → soubor `.vscode/mcp.json` → nad záznamem `km-ln` klikni na **Start**.
6. Copilot Chat → režim **Agent** → ikona nástrojů → musí tam být `km-ln` s 5 nástroji.
7. Testovací otázky:
   - „Vyhledej v knowledge, co je tdsls401.“ (funguje i bez LN)
   - „Ukaž posledních 5 prodejních zakázek z LN.“ (ln_odata_get)
   - „Kolik artiklů je v tcibd001? Udělej dotaz přes Compass.“ (compass_query)
8. Když něco nefunguje, podívej se do `logs/audit.log`. Je tam každé volání i s chybou.

**Nejčastější chyby:**

| Chyba | Příčina |
|---|---|
| `401` | špatný nebo starý `.ionapi`, případně uživatel nemá roli |
| `403` | chybí API autorizace pro danou routu v LN |
| `404` | špatný název služby nebo entity. Ověř v API Gateway → Try it out |
| Compass `FAILED` | špatný název tabulky nebo sloupce v Data Lake (sloupce mají často prefix, např. `t$orno`) |

---

## KROK 4: Rozdělení práce v týmu

| Kdo | Úkol | Soubor |
|---|---|---|
| **Ty** | tenant, servisní účet, review PR | – |
| **Štefan** | `compass_query` + data dictionary | `knowledge/dd/*.md` |
| **Tim** | `ln_odata_get`, později custom REST z lednového workshopu | `knowledge/rest-api/` |
| **Nikolaj** | `ft_ido_load` + IDO dokumentace | `knowledge/factory-track.md` |
| **Jochen** | reporty a DOM do wiki | `knowledge/reports/*.md` |
| **Claudia, Ralf, Chris** | používat, hlásit, co chybí (Issues) | – |

---

## KROK 5: Harmonogram

| Kdy | Co |
|---|---|
| **do 15. 11.** | Krok 1 + 2 + 3: běží to u tebe |
| **do konce 11.** | Štefan a Tim vyzkouší, první PR do `knowledge/` |
| **workshop v MK (začátek 12.)** | každý si to nainstaluje (README, 15 min), praktické cvičení: „vyřeš úkol z Berdingu s Copilotem a MCP“ |
| **leden 2027** | REST workshop → nové nástroje od Tima, Factory Track od Nikolaje |
| **po osvědčení** | centrální běh přes HTTP s přihlášením (Entra ID) a zvážit Eclipse pro všechny |

---

## Bezpečnostní pravidla (řekni je na workshopu nahlas)

1. **Jen testovací tenant.** Žádná produkce zákazníka.
2. `.ionapi` sdílet jen přes firemní správce hesel, **ne e-mailem ani Teams**.
3. Server je **jen pro čtení**. Zápis do LN řešíme přes Infor GenAI se schvalováním (Rampen-Rudi, Artikel-Anton).
4. V `knowledge/` nesmí být osobní údaje ani hesla.
5. Nový nástroj jen přes Pull Request.
