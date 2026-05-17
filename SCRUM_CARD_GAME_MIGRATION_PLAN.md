# Scrum Card Game → Satori-sivusto: toteutussuunnitelma

> Tämä dokumentti on **handoff-spec** Satori-nettisivua (Next.js / React, Vercel)
> rakentavalle agentille. Lukija ei ole nähnyt aiempaa keskustelua, joten tämä on
> itsenäinen ja sisältää kaiken tarvittavan kontekstin. Koodi-identifierit ovat
> englanniksi, selitykset suomeksi.

---

## 1. Tavoite lyhyesti

Siirrä olemassa oleva **Scrum Card Game** -sivusto pois nykyisestä hauraasta
GitHub Pages -toteutuksesta **osaksi uutta Satori-nettisivua** (Next.js / React,
deployataan Vercelissä). Kortit toteutetaan **interaktiivisina HTML/CSS-kortteina**
— ei upotettuina esityksinä. iCloud + Embedly -riippuvuus poistetaan kokonaan ja
sisältö self-hostataan osana Satori-sivustoa.

**Tärkeää:** Peli ei tule omaksi erilliseksi projektikseen eikä omaksi Vercel-
deployksi. Se rakennetaan **suoraan Satori-sivuston Next.js-repoon** feature-
moduulina ja deployataan osana Satori-sivuston olemassa olevaa Vercel-pipelinea.

---

## 2. Lähtötilanne (nykyinen sivusto)

Repo `teemuhyyrylainen/scrumcardgame.github.io` on staattinen GitHub Pages -sivu.
Sisältö on neljä lähes identtistä HTML-sivua:

| Tiedosto      | Sivu / pakka          | Embedly → iCloud Keynote -jakolinkki        |
|---------------|-----------------------|---------------------------------------------|
| `index.html`  | Product Owner         | `1_PO_Cards_2020_DIGI_v1`                   |
| `index2.html` | Product Backlog       | `2_BPL_Cards_2020_DIGI_v1`                  |
| `index3.html` | Scrum Master          | `3_SM_Cards_2020_DIGI_v1`                   |
| `index4.html` | Sprint Planning       | `4_PLAN_Cards_2020_DIGI_v1`                 |

Jokainen sivu sisältää:

- W3.CSS + Google Fonts (Raleway) CDN:stä,
- navigaatiopalkin neljään sivuun (aktiivinen lihavoitu `<b>`),
- yhden Embedly-kortin joka upottaa iCloud-Keynote-esityksen (`platform.js`),
- footerin tekijänoikeustekstillä.

### Nykyiset ongelmat (ratkaistaan tässä siirrossa)

1. **Hauras ulkoinen riippuvuus**: kortit ovat iCloud-Keynote-jakolinkkien takana
   ja renderöityvät Embedly-widgetillä. Ei toimi offline, ei tyylihallintaa,
   rikkoutuu jos iCloud-jako poistetaan/vanhenee.
2. **Rikkinäinen HTML-rakenne**: `<head>` on `<body>`:n sisällä, `<body>` kahdesti,
   `<p>`-tageja ei suljeta. Selaimet korjaavat, mutta merkintä on viallista.
3. **Toistettu markup**: navigaatio + footer kopioitu neljästi.
4. **`.DS_Store` committoitu** repoon.
5. Korttien varsinaista sisältöä ei voi versionhallita eikä tyylittää, koska se
   asuu iCloudissa.

### Tärkeä: tekijänoikeus / attribuutio (SÄILYTETTÄVÄ)

Uuden toteutuksen on **näytettävä** alkuperäinen attribuutio (esim. footerissa
tai About-osiossa). Älä poista tätä:

> © 2020 Scrum Card Game by **Petri Heiramo**, **AgileCraft**
> (https://agilecraft.wordpress.com, https://www.linkedin.com/in/pheiramo/).
> All rights reserved, but you can use this game freely with appropriate
> attribution.
> Online version by **Teemu Hyyryläinen**, **Reaktor**
> (https://reaktor.com/training, https://www.linkedin.com/in/teemuhoo/).

> ⚠️ Avoin kohta: varmista oikeuksien haltijalta, että pelin sisällön saa
> transformoida HTML/CSS-muotoon ja julkaista Satori-sivustolla. Attribuutio on
> joka tapauksessa pakollinen.

---

## 3. Domain-konteksti: mikä Scrum Card Game on

Scrum Card Game on Petri Heiramon koulutuspeli, jossa pelataan korteilla jotka
edustavat Scrum-tiimin toimintaa. Sivuston neljä "pakkaa" vastaavat pelin
roolikortteja/vaiheita:

- **PO** – Product Owner -kortit
- **BPL** – (Product) Backlog -kortit
- **SM** – Scrum Master -kortit
- **PLAN** – Sprint Planning -kortit

Kukin pakka on joukko kortteja. Korttien **tarkkaa sisältöä ei ole vielä saatavilla**
(asuu iCloud-Keynote-tiedostoissa; käyttäjällä on alkuperäiset, formaatti
tarkennetaan myöhemmin). Siksi tämä suunnitelma määrittelee **datamallin ja UI:n
geneerisesti**, ja oikea sisältö täytetään dataan myöhemmin ilman koodimuutoksia.

---

## 4. Kohdearkkitehtuuri (Next.js / React)

Toteutetaan **itsenäisenä feature-moduulina** Satori-repon sisällä, niin että
se on helppo pudottaa paikalleen ja teema periytyy Satori-sivuston design-
systeemistä.

### 4.1 Hakemistorakenne (ehdotus — sovita Satori-repon konventioihin)

```
src/features/scrum-card-game/
  data/
    decks.ts            # Deck-metadata (po, bpl, sm, plan)
    cards.po.ts         # PO-pakan kortit (placeholder kunnes sisältö saatu)
    cards.bpl.ts
    cards.sm.ts
    cards.plan.ts
    index.ts            # Koostaa + validoi datan
  types.ts              # Card / Deck TypeScript-tyypit
  components/
    GameLayout.tsx      # Otsikko, deck-navigaatio, footer/attribuutio
    DeckNav.tsx         # 4 pakan välilehdet/navigaatio
    DeckView.tsx        # Yhden pakan korttiruudukko
    Card.tsx            # Yksittäinen kortti, flip front/back
    CardModal.tsx       # Suurennettu kortti + näppäimistönavigaatio
    Attribution.tsx     # Tekijänoikeus (pakollinen)
  styles/
    *.module.css        # Tai Tailwind — Satori-systeemin mukaan
  scrumCardGame.routes.ts# Reititysapuri (App/Pages Router)
  README.md             # Lyhyt käyttöönotto Satori-repolle
scripts/
  import-cards.ts       # Lähdedatan → datamalli, schema-validointi
```

### 4.2 Reititys ja integraatio Satori-sivustoon

- **Base path konfiguroitava** (oletus esim. `/scrum-card-game`). Älä kovakoodaa
  domainia.
- **App Router** (oletus, varmista Satori-repon konventio):
  - `app/scrum-card-game/page.tsx` → yleisnäkymä + DeckNav
  - `app/scrum-card-game/[deck]/page.tsx` → DeckView, `deck ∈ {po,bpl,sm,plan}`
  - `generateStaticParams` neljälle pakalle (täysin staattinen, ei runtimea)
- **Pages Router -fallback** jos Satori käyttää sitä:
  - `pages/scrum-card-game/index.tsx`, `pages/scrum-card-game/[deck].tsx`
- **Deep-link yksittäiseen korttiin**: query/hash `?card=<id>` avaa CardModalin
  (jaettavat linkit yksittäisiin kortteihin).
- Lisää linkki Satori-sivuston navigaatioon (jätä agentin sovitettavaksi
  Satori-IA:han).

### 4.3 Komponenttien vastuut

- **GameLayout**: kehys, otsikko "Scrum Card Game", DeckNav, `<Attribution/>`.
- **DeckNav**: 4 pakkaa, aktiivinen korostettu, näppäimistöllä navigoitava,
  korvaa nykyisen `<b>`-pohjaisen navin.
- **DeckView**: responsiivinen korttiruudukko (CSS grid), lazy-render isoille
  pakoille, "shuffle"/"draw"-tila valinnainen (ks. avoimet kysymykset).
- **Card**: kääntyvä kortti (CSS 3D flip, `prefers-reduced-motion` huomioitu),
  front = nimi/otsikko, back = kuvaus/säännöt. Klikkaus → CardModal.
- **CardModal**: suuri kortti, ←/→ selaa pakkaa, Esc sulkee, focus-trap.
- **Attribution**: §2:n teksti linkkeineen — pakollinen, näkyvissä.

### 4.4 Ei-toiminnalliset vaatimukset

- **Saavutettavuus**: semanttinen HTML, focus management modalissa,
  `prefers-reduced-motion`, riittävät kontrastit, näppäimistöllä pelattava.
- **Responsiivisuus**: toimii mobiilista työpöytään; ruudukko skaalautuu.
- **Print-tyyli**: peli on fyysinen korttipeli → `@media print` jolla pakan saa
  tulostettua kortteina (yksi kortti / alue). Hyödyllinen oikeasti.
- **Teema**: käytä Satori-sivuston design-tokeneita / CSS-muuttujia. Älä tuo
  W3.CSS:ää. Fontti Satori-systeemistä (alkuperäinen oli Raleway — ei pakko).
- **Ei runtime-riippuvuuksia** ihannetilassa: flip puhtaalla CSS:llä. Jos
  animaatiokirjasto, pidä minimaalisena ja perustele.
- **i18n-koukku**: alkuperäinen on englanniksi. Jos Satori on monikielinen,
  pidä korttitekstit datassa lokalisoitavissa (ks. datamalli).

---

## 5. Datamalli

Sisältö erotetaan koodista täysin. Tyypit (`types.ts`):

```ts
export type DeckId = 'po' | 'bpl' | 'sm' | 'plan';

export interface Deck {
  id: DeckId;
  /** Näytettävä nimi, esim. "Product Owner" */
  name: string;
  /** Lyhyt kuvaus pakasta */
  description?: string;
  /** Lähde-Keynote tunnistetta varten, esim. "1_PO_Cards_2020_DIGI_v1" */
  sourceRef?: string;
  /** Teema/väri Satori-tokeneilla */
  accent?: string;
}

export interface Card {
  /** Vakaa, URL-turvallinen id (deep-linkkaukseen), esim. "po-01" */
  id: string;
  deck: DeckId;
  /** Järjestys pakassa */
  order: number;
  /** Kortin etupuolen otsikko */
  title: string;
  /** Etupuolen alaotsikko/rooli (valinnainen) */
  subtitle?: string;
  /** Takapuolen sisältö: kuvaus / säännöt. Markdown sallittu. */
  body: string;
  /** Vapaa luokittelu, esim. "event" | "role" | "artifact" */
  category?: string;
  /** Pisteet/arvo jos kortilla sellainen */
  value?: number;
  /** Valinnainen kuva (jos sisältö tuodaan kuvina, ei pelkkänä HTML) */
  image?: { src: string; alt: string };
  notes?: string;
}
```

- `data/index.ts` koostaa pakat + kortit, ja **validoi** (zod tms. jos Satori
  käyttää; muuten kevyt runtime-tarkistus): uniikit `id`:t, `deck` viittaa
  olemassa olevaan pakkaan, `order` järjestää.
- Aloita **placeholder-korteilla** (esim. 3–5 / pakka) jotka noudattavat tätä
  rakennetta, selkeällä `TODO: replace with real content from <sourceRef>`.
- i18n-variantti tarvittaessa: `title`/`body` → `Record<Locale,string>` tai
  erillinen käännöstiedosto. Päätä Satori-i18n:n mukaan.

---

## 6. Sisältöputki (alkuperäisistä Keynote-tiedostoista dataan)

Alkuperäiset kortit ovat Keynote-tiedostoissa (`1_PO…`, `2_BPL…`, `3_SM…`,
`4_PLAN…`). **Lähdeformaatti ja toimitustapa tarkennetaan myöhemmin** — datamalli
on suunniteltu niin ettei se estä etenemistä.

Suositeltu prosessi kun sisältö saadaan:

1. **Vie Keynote** → joko (a) teksti talteen (yksi dia = yksi kortti) tai
   (b) PNG/SVG/PDF per dia.
2. **Ensisijainen tavoite (käyttäjän valinta): aidot HTML/CSS-kortit** →
   transkriboi korttien teksti `data/cards.*.ts`-tiedostoihin datamallin
   mukaan. Kuva-vienti voi toimia välivaiheen lähteenä/tarkistuksena.
3. **`scripts/import-cards.ts`**: ottaa strukturoidun lähteen (CSV/JSON),
   tuottaa/validioi datatiedostot schemaa vasten. Toimita CSV/JSON-täyttöpohja
   sarakkeilla: `deck,order,id,title,subtitle,body,category,value`.
4. Säilytä `sourceRef` jokaisessa pakassa jäljitettävyyttä varten.

> Kun lopullinen formaatti tiedetään, tämä putki konkretisoidaan. Siihen asti
> rakenne ja UI rakennetaan placeholder-datalla.

---

## 7. Vercel-deployment

- Peli shippaa **osana Satori-sivuston Next.js-sovellusta** → ei omaa Vercel-
  projektia, ei erillistä konfiguraatiota. Se deployautuu Satori-sivuston
  olemassa olevassa Vercel-pipelinessa automaattisesti.
- Sivut ovat staattisia (SSG, `generateStaticParams`), joten ei runtime- eikä
  ympäristömuuttujatarvetta.
- **Vanha sivusto**: kun uusi on tuotannossa, harkitse `scrumcardgame.github.io`
  → uusi Satori-URL -uudelleenohjaus (tai päivitä linkit). Tämä on erillinen,
  myöhempi tehtävä eikä kuulu Satori-agentin scope-alkuun.

---

## 8. Migraatiovaiheet (järjestyksessä)

1. **Vahvista** alkuperäisten korttien lähde + vientiformaatti (avoin).
2. **Vahvista Satori-repon konventiot**: App vs Pages Router, design system
   (Tailwind / CSS Modules / muu), monorepo vai ei, pelin URL/base path,
   monikielisyys.
3. **Scaffoldaa** feature-moduuli (§4.1) tyypeillä + placeholder-datalla.
4. **Rakenna komponentit**: GameLayout, DeckNav, DeckView, Card (flip),
   CardModal, Attribution.
5. **Wire reitit** Satori-Next.js-sovellukseen + lisää linkki Satori-naviin.
6. **Tyylitä** Satori-design-systeemillä, poista W3.CSS-tyyppinen riippuvuus.
7. **A11y + print + responsiivisuus**.
8. **Täytä oikea korttisisältö** (§6) kun lähde saatu.
9. **QA selaimessa**: golden path (selaa pakat, käännä kortti, modal, deep-link)
   + reunatapaukset (tyhjä pakka, pitkä teksti, mobiili, reduced-motion, print).
10. **Vanhan sivuston uudelleenohjaus** (erillinen jälkitehtävä).

Vaiheet 3–7 voi tehdä heti placeholder-datalla; vaihe 8 odottaa sisältöä.

---

## 9. Avoimet päätökset / tarvittavat syötteet

Satori-agentti tarvitsee nämä (osa ratkaistaan käyttäjän kanssa):

- **Korttisisällön lähdeformaatti** ja toimitustapa (Keynote-vienti: teksti vs.
  PDF/PNG/SVG). — *Avoin, käyttäjä tarkentaa.*
- **Satori-repon stack-yksityiskohdat**: App vai Pages Router; Tailwind vai
  CSS Modules vai muu; monorepo-rakenne; pelin lopullinen polku/URL.
- **Monikielisyys**: onko Satori monikielinen → vaikuttaa datamalliin (§5 i18n).
- **Pelilogiikan laajuus**: halutaanko pelkkä korttiselain/-referenssi (kuten
  nykyinen), vai myös interaktiivista pelimekaniikkaa (sekoitus, jako, kierrokset,
  pisteet, ajastin)? Nykyinen sivusto on pelkkä esityskatselin — oletus tässä on
  selattava/jaettava korttinäkymä, mekaniikka erikseen sovittavissa.
- **Lisensointivahvistus** sisällön transformointiin (attribuutio joka
  tapauksessa pakollinen, §2).

---

## 10. Tiivistetty toimeksianto Satori-agentille

> Lisää uuteen Satori Next.js -sivustoon itsenäinen `scrum-card-game` feature-
> moduuli: 4 pakkaa (PO, BPL, SM, PLAN), sisältö koodista erotettuna typed
> datamalliin (§5), interaktiiviset CSS-flip-kortit + modal + deep-link,
> staattiset reitit `/scrum-card-game` ja `/scrum-card-game/[deck]`, Satori-
> design-systeemin teema, a11y + print + responsiivisuus, ei iCloud/Embedly-
> riippuvuutta. Aloita placeholder-datalla; oikea korttisisältö täytetään
> myöhemmin (§6) lähdeformaatin varmistuttua. Säilytä §2:n attribuutio
> näkyvissä. Deployaa osana Satori-sivuston olemassa olevaa Vercel-pipelinea
> — ei erillistä projektia.
