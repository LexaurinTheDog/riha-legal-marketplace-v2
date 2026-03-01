# AML Prověření klienta

Skill pro komplexní AML prověřování smluvních stran dle zákona č. 253/2008 Sb. (AML zákon) se zaměřením na advokátní praxi a podnikatele.

## Spuštění

Trigger: `/aml-provereni`, `AML kontrola`, `prověř klienta`, `identifikace klienta AML`

## Parametry

- `klient` - Název/jméno klienta k prověření (povinný)
- `ico` - IČO právnické osoby (volitelný)
- `typ` - Typ klienta: `fo` (fyzická osoba), `po` (právnická osoba), `podnikatel` (volitelný)

## Workflow prověření

### 1. IDENTIFIKACE KLIENTA

#### Fyzická osoba:
- [ ] Jméno a příjmení
- [ ] Datum narození / rodné číslo
- [ ] Státní občanství
- [ ] Trvalý pobyt
- [ ] Typ a číslo průkazu totožnosti
- [ ] Ověření totožnosti z průkazu (kopie)

#### Právnická osoba:
- [ ] Obchodní firma
- [ ] IČO
- [ ] Sídlo
- [ ] Právní forma
- [ ] Zápis v obchodním rejstříku
- [ ] Předmět podnikání
- [ ] Identifikace osoby jednající za PO (jednatel, prokurista)

### 2. OVĚŘENÍ VE VEŘEJNÝCH REGISTRECH

Použij kombinaci MCP Brave, MCP Perplexity a WebFetch pro maximální pokrytí:

**Strategie použití nástrojů:**

- **MCP Perplexity `perplexity_research`** – primární nástroj pro hloubkový průzkum osoby/firmy. Zadej dotaz v češtině s plným jménem, IČO, adresou. Vrací strukturované výsledky s citacemi. Použij pro: ekonomické vazby, mediální zmínky, soudní spory, insolvence, exekuce.
- **MCP Perplexity `perplexity_search`** – rychlé vyhledání konkrétních faktů (IČO, sankce, PEP status). Použij pro: ověření údajů z registrů, dohledání konkrétních informací.
- **MCP Perplexity `perplexity_ask`** – konverzační dotazy pro interpretaci nálezů. Použij pro: vyhodnocení rizikových indikátorů, právní klasifikaci zjištěných skutečností.
- **MCP Perplexity `perplexity_reason`** – analytické úlohy. Použij pro: celkové hodnocení rizikového profilu, posouzení ekonomické logiky transakce, identifikaci red flags.
- **MCP Brave `brave_web_search`** – doplňkové webové vyhledávání, zejména pro české registry a portály.
- **WebFetch** – přímý přístup k API registrů (ARES REST API, ISIR, kurzy.cz, Hlídač státu).

**Doporučený postup:**

1. Nejprve spusť `perplexity_research` s dotazem: *"Prověř ekonomické vazby, firmy, insolvence, exekuce a mediální zmínky osoby [JMÉNO], narozena [DATUM], bytem [ADRESA]. Uveď všechny firmy kde figuruje jako jednatel, společník nebo člen orgánů, včetně historických."* (pro FO) nebo *"Prověř firmu [NÁZEV], IČ [IČO], sídlo [ADRESA]. Uveď jednatele, společníky, skutečné majitele, předmět podnikání, historii změn názvů, propojené firmy, účetní závěrky, mediální zmínky, soudní spory."* (pro PO)
2. Paralelně spusť WebFetch na ARES REST API a ISIR.
3. Výsledky z Perplexity doplň a ověř přes Brave search a WebFetch na konkrétních registrech (kurzy.cz, Hlídač státu, justice.cz).
4. Pro vyhodnocení všech nálezů použij `perplexity_reason` s úlohou: *"Na základě těchto zjištění vyhodnoť rizikový profil klienta z hlediska AML: [shrnutí nálezů]. Identifikuj red flags a doporuč rizikovou kategorii."*

Prověř v následujících registrech:

#### ARES (Administrativní registr ekonomických subjektů)
- REST API: `https://ares.gov.cz/ekonomicke-subjekty-v-be/rest/ekonomicke-subjekty/{ICO}` (JSON)
- Web: `https://ares.gov.cz/ekonomicke-subjekty?ico={ICO}`
- Ověř: existenci subjektu, sídlo, právní formu, předmět podnikání
- **Fallback:** Pokud ARES REST API vrátí 404, použij `rejstrik-firem.kurzy.cz/{ICO}/`

#### Obchodní rejstřík (Justice.cz)
- URL: https://or.justice.cz/
- Ověř: statutární orgán, způsob jednání, základní kapitál, společníky
- Stáhni relevantní dokumenty ze sbírky listin (stanovy, účetní závěrky)
- **Fallback:** `rejstrik-firem.kurzy.cz/{ICO}/` – spolehlivý zdroj OR dat

#### Evidence skutečných majitelů
- URL: https://esm.justice.cz/ (přístup přes datovou schránku nebo notáře)
- Zjisti skutečného majitele (UBO - Ultimate Beneficial Owner)
- Ověř: jméno, příjmení, datum narození, státní občanství, podíl na hlasovacích právech

#### Insolvenční rejstřík
- REST API: `https://isir.justice.cz:8443/isir_ciks_ws/IsirWsCiksService?ico={ICO}` (pro PO)
- REST API: `https://isir.justice.cz:8443/isir_ciks_ws/IsirWsCiksService?rc={RC}` (pro FO – bez lomítka)
- Web: `https://isir.justice.cz/isir/ueu/vysledek_lustrace.do?ic={ICO}` nebo `rc={RC}`
- Prověř, zda klient není v insolvenci
- **Tip:** Vždy ověřit i FO jednatele PO, nejen samotnou PO

#### Hlídač státu (klíčový zdroj pro FO)
- URL: `https://www.hlidacstatu.cz/osoba/{jmeno-prijmeni-rok-narozeni}`
- Nejlepší zdroj pro kompletní přehled ekonomických vazeb fyzické osoby
- Zobrazuje: všechny firmy (aktivní i historické), funkce, období, vazby na veřejnou správu
- **Tip:** U běžných jmen ověřit správnou osobu podle roku narození a adresy

#### Živnostenský rejstřík (RŽP)
- URL: `https://www.rzp.cz/`
- Ověř registrace OSVČ – osoba může mít **více IČO** (různá sídla podnikání)

#### Další fungující zdroje
- `rejstrik-firem.kurzy.cz/{ICO}/` – spolehlivý, rychlý, detailní výpis z OR
- `www.penize.cz` – vyhledání osoby/firmy, ekonomické vazby
- `www.podnikatel.cz` – vyhledání podnikatelů

### 2b. PROVĚŘENÍ EKONOMICKÝCH VAZEB (zejména FO)

Pro každou fyzickou osobu prověř kompletní ekonomické vazby. Postup ověřený v praxi (případ Červinka, 02/2026):

#### Metodika:
1. **Hlídač státu** (`hlidacstatu.cz/osoba/`) – primární zdroj, zobrazí všechny firmy naráz
2. **Perplexity research** – hloubkový průzkum s dotazem na všechny ekonomické vazby
3. **ARES/kurzy.cz** – ověření detailů jednotlivých firem (IČ, sídlo, předmět podnikání, ZK)
4. **Brave search** – doplňkové vyhledání na penize.cz, podnikatel.cz

#### Mapuj tyto kategorie vazeb:

**A) Aktivní ekonomické vazby:**
- [ ] Funkce jednatele/člena statutárního orgánu (aktivní společnosti)
- [ ] Společnické podíly (aktivní společnosti)
- [ ] U každé firmy zjisti: IČ, sídlo, spisovou značku, datum vzniku, ZK, předmět podnikání, společníky, způsob jednání
- [ ] Ověř, zda firma vykazuje reálnou ekonomickou činnost (web, zaměstnanci, obrat)

**B) Registrace OSVČ:**
- [ ] Živnostenský rejstřík – osoba může mít **více IČO** pod různými sídly
- [ ] Sídlo podnikání vs. trvalý pobyt – může se lišit

**C) Historické ekonomické vazby:**
- [ ] Ukončené funkce v obchodních společnostech
- [ ] Zaniklé společnosti / společnosti v likvidaci
- [ ] Kategorizuj: investiční/finanční, obchodní/průmyslové, realitní, jiné
- [ ] Časový rozsah podnikatelské aktivity (od–do)

**D) Nepřímé vazby:**
- [ ] Společnosti, ve kterých figurovaly firmy prověřované osoby jako společníci
- [ ] Řetězení vlastnických struktur

**E) Rodinné vazby:**
- [ ] Shodné příjmení + bydliště v jiných firmách = pravděpodobný příbuzenský vztah
- [ ] Manžel/ka jako společník/jednatel v propojených firmách
- [ ] Dokumentuj, ale nehodnoť automaticky jako red flag

**F) Vazby na veřejnou správu:**
- [ ] Registr smluv (Hlídač státu) – smluvní vztahy firem s veřejným sektorem
- [ ] Dotace, veřejné zakázky

#### Struktura výstupní zprávy o ekonomických vazbách:

Zprávu strukturuj do těchto sekcí (ověřený formát):
1. **Identifikace prověřované osoby** – jméno, r.č., datum narození, trvalý pobyt, postavení v transakci
2. **Metodika prověření** – seznam použitých registrů a databází
3. **Aktivní ekonomické vazby** – detailní výpis aktivních firem s kompletními údaji z OR
4. **Registrace jako OSVČ** – všechna IČO, sídla podnikání
5. **Historické ekonomické vazby** – kategorizované (investiční, obchodní, realitní)
6. **Nepřímé vazby** – vazby přes firmy
7. **Vazby na veřejnou správu** – registr smluv
8. **Kontroly ve veřejných registrech** – ISIR, sankce, PEP
9. **Souhrnné hodnocení** – profil osoby, obory činnosti, rodinné vazby, vyhodnocení rizik, riziková kategorie

### 3. KONTROLA PEP (Politicky exponovaná osoba)

Ověř, zda klient nebo skutečný majitel není PEP podle § 4 odst. 5 AML zákona:

- [ ] Člen vlády, poslanec, senátor
- [ ] Člen Evropského parlamentu
- [ ] Vedoucí představitel státní správy
- [ ] Člen nejvyššího soudu, ústavního soudu
- [ ] Člen vedení centrální banky
- [ ] Vysoký důstojník ozbrojených sil
- [ ] Člen řídícího orgánu státního podniku
- [ ] Osoba blízká PEP

**Zdroje pro ověření:**
- PEP check: https://www.pepcheck.cz/
- AML solutions: https://www.amlsolutions.cz/politicky-exponovana-osoba
- Metodický pokyn FAÚ č. 7

### 4. KONTROLA MEZINÁRODNÍCH SANKCÍ

Ověř, zda klient není na sankčním seznamu:

- [ ] EU Sanctions Map: https://www.sanctionsmap.eu/
- [ ] Národní sankční seznam ČR
- [ ] Seznam OSN

**U právnických osob ověř také:**
- Skutečné majitele
- Členy statutárních orgánů
- Osoby z vlastnické a řídící struktury

### 5. PŮVOD FINANČNÍCH PROSTŘEDKŮ (zejména u podnikatelů)

#### Zjisti a dokumentuj:

**A) Zdroj prostředků:**
- [ ] Příjmy z podnikání - ověř z účetních závěrek, daňových přiznání
- [ ] Příjmy ze zaměstnání
- [ ] Prodej majetku (nemovitosti, podíly)
- [ ] Dědictví, dar
- [ ] Úvěr/hypotéka (doložit smlouvou)
- [ ] Výnosy z investic

**B) U podnikatelů zvláště prověř:**
- [ ] Účetní závěrky za poslední 3 roky (ze sbírky listin OR)
- [ ] Obrat vs. částka transakce - je přiměřená?
- [ ] Předmět podnikání vs. účel transakce - dává smysl?
- [ ] Cash flow - má podnikatel reálné příjmy?
- [ ] Existence podnikatelské činnosti (webové stránky, reference, zaměstnanci)

**C) Red flags - podezřelé indikátory:**
- Částka neodpovídá běžným příjmům/obratu
- Klient odmítá doložit původ prostředků
- Složité vlastnické struktury bez ekonomického opodstatnění
- Zahraniční struktury v rizikových jurisdikcích
- Rychlé převody bez zjevného důvodu
- Hotovostní transakce nad limit

### 6. HODNOCENÍ RIZIKA

Kategorizuj klienta podle rizikového profilu:

#### Nízké riziko:
- Český státní občan
- Transparentní vlastnická struktura
- Obvyklý obchod odpovídající profilu
- Prostředky z ověřitelných zdrojů
- Aktivní firmy vykazují reálnou ekonomickou činnost (web, zaměstnanci, obrat)
- Vyšší počet historických firem v likvidaci odpovídá profilu dlouhodobě aktivního podnikatele (např. 10+ firem za 25 let ≠ red flag)
- Žádné záznamy v ISIR, na sankčních seznamech

#### Střední riziko:
- Složitější vlastnická struktura
- Vyšší hodnota transakce
- První obchod s klientem
- **SPV struktura u developera** – projektová společnost (SPV) je běžná praxe, ale ověř: minimální ZK, krátká historie, shodné osoby v mateřské i dceřiné společnosti
- **Časté přejmenovávání společnosti** (3+ názvy za krátké období) – může indikovat změnu účelu, ale i legitimní restrukturalizaci
- **Jednatel bez společnického podílu** – osoba řídí firmu, ale nevlastní ji → ověř vztah k vlastníkovi

#### Vysoké riziko:
- PEP nebo osoba blízká PEP
- Zahraniční struktura
- Rizikové jurisdikce (offshore)
- Nesrovnalosti v dokumentaci
- Neochota doložit původ prostředků
- Transakce neodpovídá ekonomickému profilu (např. kupní cena výrazně převyšuje zjištěné obraty/příjmy)

#### Pravidla hodnocení (ověřená praxe):
- **Kontext je klíčový:** Vyšší počet firem v likvidaci ≠ automaticky red flag. Posuď v kontextu délky podnikatelské kariéry a charakteru firem (projektové/investiční společnosti se zakládají a ruší častěji).
- **Rodinné vazby:** Manžel/ka jako společník/jednatel v propojené firmě je běžná praxe, dokumentuj ale nepenalizuj.
- **Více IČO u OSVČ:** Neobvyklé, ale legitimní (různá sídla podnikání).
- **Ekonomická logika transakce:** Posuď, zda výše transakce odpovídá zjištěnému ekonomickému profilu (obory, počet aktivních firem, typ činnosti → schopnost generovat odpovídající příjmy).

### 7. VÝSTUP - AML DOKUMENTACE

Vytvoř kompletní AML dokumentaci:

1. **Identifikační a kontrolní formulář** - použij vzor z:
   `/Users/vojte/Dropbox/Vojtech/Obsidian_RIHA_legal/RIHA legal/Template/AML_formular_identifikace_klienta_vzor.md`

2. **Záznam o prověření** obsahující:
   - Datum a způsob identifikace
   - Výsledky kontrol ve veřejných registrech
   - PEP status
   - Sankční status
   - Zdroj prostředků
   - Hodnocení rizika
   - Doporučení (pokračovat/odmítnout/zesílená kontrola)

3. **Přílohy:**
   - Kopie průkazu totožnosti
   - Výpis z OR/ARES
   - Výpis z evidence skutečných majitelů
   - Doklady o původu prostředků

## Registry a nástroje

| Registr/Nástroj | URL | Účel | Spolehlivost |
|-----------------|-----|------|--------------|
| ARES REST API | `ares.gov.cz/ekonomicke-subjekty-v-be/rest/ekonomicke-subjekty/{ICO}` | Základní info o subjektech (JSON) | Střední (občas 404) |
| Obchodní rejstřík | https://or.justice.cz/ | Detaily PO, sbírka listin | Vysoká |
| **kurzy.cz** | `rejstrik-firem.kurzy.cz/{ICO}/` | **Nejspolehlivější fallback pro OR/ARES** | **Vysoká** |
| **Hlídač státu** | `hlidacstatu.cz/osoba/{jmeno-prijmeni}` | **Kompletní ekonomické vazby FO** | **Vysoká** |
| Evidence skutečných majitelů | https://esm.justice.cz/ | UBO | Omezená (vyžaduje DS) |
| Insolvenční rejstřík | https://isir.justice.cz/ | Insolvence | Vysoká |
| Živnostenský rejstřík | https://www.rzp.cz/ | OSVČ registrace | Vysoká |
| EU Sanctions Map | https://www.sanctionsmap.eu/ | Sankce EU | Nízká (JS rendering) |
| PEP check | https://www.pepcheck.cz/ | PEP ověření | Střední |
| Katastr nemovitostí | https://nahlizenidokn.cuzk.cz/ | Vlastnictví nemovitostí | Vysoká |
| penize.cz | https://www.penize.cz/ | Vyhledání osob a firem | Střední |
| podnikatel.cz | https://www.podnikatel.cz/ | Vyhledání podnikatelů | Střední |

**Poznámky ke spolehlivosti (ověřeno 02/2026):**
- ARES REST API občas vrací 404 i pro existující subjekty → vždy mít fallback na kurzy.cz
- OR Justice.cz – přímé URL na firmu mohou být neplatné → použij kurzy.cz jako alternativu
- EU Sanctions Map – JS-rendered stránka, WebFetch nezíská obsah → použij Brave search pro sankční info
- Hlídač státu – nejlepší jednotný zdroj pro FO, ale u běžných jmen ověř identitu podle roku narození/adresy
- ISIR – spolehlivý, ale hledej i FO jednatele, nejen samotnou PO

## Právní rámec

- **Zákon č. 253/2008 Sb.** - AML zákon
- **Zákon č. 37/2021 Sb.** - o evidenci skutečných majitelů
- **Zákon č. 69/2006 Sb.** - o provádění mezinárodních sankcí
- **Usnesení ČAK č. 2/2008** - postup advokátů
- **Metodické pokyny FAÚ** - zejména č. 7 (PEP)

## Archivace

Dokumentaci uchovávej **10 let** od uskutečnění obchodu nebo ukončení obchodního vztahu (§ 16 AML zákona).

---

**Poznámka:** Tento skill slouží jako průvodce AML prověřením. Vždy postupuj s ohledem na konkrétní okolnosti případu a rizikovost klienta. V případě podezřelého obchodu je nutné oznámení ČAK (§ 18 AML zákona).
