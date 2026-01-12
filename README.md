# ELT sistem za analizu kvaliteta vazduha u Evropi

**(Batch + Near-Real-Time obrada podataka – OpenAQ)**

---

## 1. Domen projekta

Ovaj projekat se bavi **analizom kvaliteta vazduha** na teritoriji Evrope, sa fokusom na zagađivače koji imaju direktan uticaj na zdravlje ljudi i životnu sredinu. Podaci obuhvataju merenja koncentracija čestica i gasova (PM2.5, PM10, NO₂, O₃, SO₂, CO...) prikupljenih sa velikog broja mernih stanica raspoređenih širom evropskih država.

Projekat kombinuje:

* **istorijske (batch) podatke velikog obima** i
* **near-real-time podatke** koji se periodično dobavljaju putem javnog API-ja,

kako bi se omogućila sveobuhvatna analitika, od dugoročnih trendova do praćenja aktuelnih stanja.

---

## 2. Motivacija

Kvalitet vazduha je u poslednjim godinama postao izuzetno značajna tema, naročito u zemljama regiona. U Srbiji, naročito tokom zimskih meseci, zabeleženi su česti i dugotrajni periodi izrazito lošeg kvaliteta vazduha, što je dovelo do povećanog interesovanja javnosti, stručnjaka i institucija (šalim se - institucije ne postoje) za dostupne podatke i njihovu analizu.

Lična motivacija za ovaj projekat proistekla je upravo iz tog konteksta — želje da se:

* razumeju obrasci zagađenja,
* uporede lokalni podaci sa širim evropskim kontekstom,
* i izgradi sistem sposoban da obradi i istorijske i aktuelne podatke na moderan i skalabilan način.

---

## 3. Cilj projekta

Glavni cilj projekta je **dizajniranje i implementacija modernog, elegantnog i skalabilnog ELT (Extract-Load-Transform) pipeline-a**, koji objedinjuje:

* **batch obradu velikih istorijskih skupova podataka**,
* **near-real-time obradu tokova podataka**,
* **idempotentno i ponovljivo učitavanje**,
* jasnu separaciju slojeva (raw / transform / curated),
* i analitički spremne podatke za krajnje korisnike.

---

## 4. Skupovi podataka i strategija obrade

### 4.1 Batch (istorijski) podaci

Primarni skup podataka čine **istorijski podaci o kvalitetu vazduha za evropske države**, preuzeti iz **OpenAQ Data Archive** repozitorijuma.

Karakteristike:

* Ukupna veličina: **nešto više od 2 GB kompresovanih CSV fajlova**
* Podaci su organizovani hijerarhijski po folderima:

  ```
  država / lokacija / godina / mesec
  ```
* Svaki fajl sadrži merenja za jednu lokaciju u okviru određenog vremenskog perioda.

Struktura zapisa u CSV fajlovima:

```text
location_id | sensors_id | location | datetime | lat | lon | parameter | units | value
```

Gde:

* `location_id` – jedinstveni identifikator merne lokacije
* `sensors_id` – identifikator senzora
* `datetime` – vreme merenja (UTC)
* `lat`, `lon` – geografske koordinate
* `parameter` – tip zagađivača
* `units` – jedinica mere
* `value` – izmerena vrednost

Najčešći parametri:

* `pm25` – PM2.5 čestice
* `pm10` – PM10 čestice
* `no2` – azot-dioksid
* `o3` – ozon
* `so2` – sumpor-dioksid
* `co` – ugljen-monoksid

Za svaku lokaciju i senzor, podaci tipično sadrže **24 merenja dnevno**, tj. **jedno merenje po satu**, što omogućava detaljnu vremensku analizu.
Ovi podaci biće obogaćeni ostalim dostupnim podacima u okviru OpenAQ API-ja, a to bi bile dodatne informacije o: državama, gradovima, instrumentima itd.

Cilj batch obrade je:

* čišćenje i normalizacija podataka,
* efikasnije skladištenje (npr. Parquet),
* i priprema podataka za analitičke upite velikog obima.

---

### 4.2 Real-time (near-real-time) podaci

Sekundarni skup podataka dobavlja se korišćenjem **OpenAQ REST API-ja**, uz zvaničnu Python biblioteku (`openaq`, verzija 0.6.0).

Važne karakteristike API-ja:

* Rate limit: **2000 zahteva po satu**
* Podaci se dobavljaju putem **polling mehanizma**
* Ne postoji push / streaming endpoint

Zbog ovih ograničenja:

* **nije moguće pratiti sve evropske lokacije u realnom vremenu**
* bira se **reprezentativan skup najvažnijih lokacija**, npr:

  * lokacije u Srbiji
  * ili najzagađenije lokacije u Evropi (na osnovu batch analize)

Podaci se dobavljaju periodično (npr. na 5 ili 10 minuta), čime se postiže **near-real-time obrada**.

Važno je istaći sledeće principe:

> **Batch obrada obezbeđuje potpunu istorijsku pokrivenost podataka za celu Evropu, dok se real-time ingest fokusira na odabrane lokacije sa najvećim analitičkim značajem, kako bi se omogućila obrada sa niskom latencijom.**

> **Merenja kvaliteta vazduha nastaju asinhrono, u heterogenim mrežama senzora. Zbog toga podaci za isti vremenski interval ne stižu istovremeno, već postepeno. Sistem obrađuje svako merenje kao nezavisan događaj i kontinuirano ažurira agregacije koristeći event-time windowing.**

> **Sistem vrši near-real-time ingest i obradu podataka, pri čemu se svako novo merenje obrađuje odmah po dostupnosti, uprkos diskretnim i provajder-zavisnim intervalima merenja.**

---

## 5. Analitička pitanja

### 5.1 Pitanja za paketnu (batch) obradu podataka

1. Kako se prosečne godišnje vrednosti određenog zagađivača menjaju po državama Evrope u posmatranom periodu?
2. Koje lokacije imaju najveći broj dana sa prekoračenjem dozvoljenih granica zagađenja?
3. Koji su sezonski obrasci zagađenja (po mesecima) za glavne gradove Evrope?
4. Da li se uočava dugoročni trend poboljšanja ili pogoršanja kvaliteta vazduha po regionima?
5. Koliko često i koliko dugo traju epizode visokog zagađenja (uzastopni sati ili dani iznad definisanog praga) po lokacijama u Evropi?
6. Koje lokacije imaju najveći broj kratkotrajnih, ali veoma visokih skokova zagađenja (npr. satna merenja višestruko veća od uobičajenih vrednosti za tu lokaciju)?
7. Kako se razlikuju nivoi zagađenja između urbanih i manje urbanizovanih sredina?
8. U kakvoj su korelaciji određeni tip zagađivač i temperatura vazduha?
9. Kako izgleda mesečni i godišnji prosek zagađenja po lokacijama?
10. Kakva je korelacija između koncentracija PM2.5 i PM10 na istim lokacijama u toku vremena?

---

### 5.2 Pitanja za obradu podataka u realnom vremenu

1. Koje lokacije beleže nagli porast određenog zagađivača u poslednjih sat vremena?
2. Prikazati lokacije i njihove senzore, koje su nedavno prestale da šalju ažuriranja merenja.
3. Da li trenutno merenje značajno odstupa od istorijskog proseka za istu lokaciju i period?
4. Kako se prosečna koncentracija zagađivača menja u toku dana na praćenom senzoru?
5. Koliko aktivnih lokacija i senzora trenutno doprinosi podacima za grad/državu?

