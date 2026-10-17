# Slim oefenen — de scheduling-logica

Technische beschrijving van de planning achter "Slim oefenen". Alles staat in
`index.html`, sectie `// ---------- Slim oefenen: spaced repetition ----------`
(regel 2459–2618).

## 1. De staat per woord

Per *item* én per *richting* (`fwd` / `rev`) wordt één object bijgehouden
(`index.html:1180-1184`):

| veld | betekenis |
|---|---|
| `g` | aantal goede antwoorden (teller, valt flink terug bij een fout) |
| `f` | aantal rondes achter elkaar fout |
| `s` | het **rondenummer** waarvoor het woord staat ingepland |
| `x` | optioneel; `true` = "niet meer oefenen" |

Daarnaast één teller per richting: `roundNumber = { fwd, rev }`. Dat is de klok van het
hele systeem — **er wordt nergens met wall-clock tijd gerekend**, alleen met rondes. De
enige timestamp in de opslag is `lastRound.at`, puur cosmetisch voor de resume-banner.

De teller loopt op in `showEndScreen()` (`index.html:2348`), dus aan het *eind* van elke
afgeronde ronde. Een afgebroken ronde (via het logo terug naar start) telt niet mee.

Opslag: `localStorage`, key `woordjesleren:progress:v1`, `wordStats` geserialiseerd als
`[itemKey, fwd|null, rev|null]` met `itemKey = "lang|word|nl"` (`index.html:1214`). `x`
wordt alleen weggeschreven als het `true` is (`withExcluded`, 1220).

Er is geen migratiecode: `loadStoredState()` (1250) eist precies de velden die
`persistState()` wegschrijft, en geeft anders `null` terug — dan begint de app schoon.
Daardoor mag elke lezer ervan uitgaan dat `roundNumber`, `lastRound` en `fileNames`
compleet zijn.

## 2. Het interval: Fibonacci, afgetopt op 10

```js
const SMART_MAX_INTERVAL = 10;
function intervalFor(goed) { return Math.min(fibonacci(goed + 2), SMART_MAX_INTERVAL); }
```

| `g` | 0 | 1 | 2 | 3 | 4 | 5 | 6+ |
|---|---|---|---|---|---|---|---|
| interval (rondes) | 1 | 2 | 3 | 5 | 8 | 10 | 10 |

Interval 1 hoort dus bij `g = 0` — en dat komt in de praktijk alleen voor bij een woord dat
nog nooit beantwoord is. Zodra je er één keer goed op antwoordt is het interval minimaal 2.

`g = 5` is de eerste stand met het maximale interval, en daarmee de grens die de selectie
gebruikt om "goed gekend" van "nog in opbouw" te scheiden (`SMART_KNOWN_GOED`, §5).

## 3. De overgang na een antwoord — `nextState` (2488)

```js
if (correct) {
  const goed = state.g + 1;
  return withExcluded({ g: goed, f: 0, s: round + intervalFor(goed) }, state.x);
}
return { g: Math.min(Math.floor(state.g / 2.5), 2), f: state.f + 1, s: round + 1 };
```

**Goed:** `g+1`, foutenteller op 0, opnieuw ingepland op `huidige ronde + interval(nieuwe g)`.
Let op: het interval wordt berekend met de *nieuwe* `g`, dus een nieuw woord dat je in ronde
R goed hebt komt terug in R+2 — het slaat precies één ronde over. `x` blijft behouden.

**Fout:** `s = round + 1`, dus **gegarandeerd in de volgende ronde terug**. `f` loopt op. `g`
valt harder terug dan de helft (`floor(g / 2.5)`) en komt nooit hoger uit dan 2 — en
`withExcluded` wordt hier bewust *niet* gebruikt, zodat een fout antwoord het woord automatisch
uit "niet meer oefenen" haalt.

Die `s = round + 1` doet **dubbel dienst**. Voor elk woord met `f > 0` geldt daardoor
`s = (de ronde waarin het voor het laatst getoond is) + 1`: de planning is er, maar het is
tegelijk de recency-stempel waar de selectie de foute woorden op sorteert (§5). Verander je
deze formule, verander dan die sortering mee.

De terugval is bewust streng. Een fout op een goed gekend woord (`g = 20`) zet je niet terug
op 10 en ook niet op de helft, maar op 2 — en dat getal is gekozen tegen
`SMART_KNOWN_GOED = 5` aan: na één fout staat een woord gegarandeerd in de groep "nog in
opbouw" die bij de selectie voorrang krijgt (§5). Er zijn dan drie goede beurten nodig
(2 → 3 → 4 → 5) om weer op het maximale interval van 10 te komen. Kwam het woord van `g ≤ 2`,
dan valt het helemaal terug naar 0, en telt het na de eerstvolgende goede beurt ook mee als
wankel woord voor de focus-rem (`g = 1`, §5).

| `g` vóór | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 en hoger |
|---|---|---|---|---|---|---|---|---|---|
| `g` erna | 0 | 0 | 0 | 1 | 1 | 2 | 2 | 2 | 2 |

`f` komt daar bovenop: dat is wat de prioriteit in de foutenpool bepaalt (§5), terwijl `g`
bepaalt hoe lang het duurt voordat het woord weer met rust gelaten wordt.

De boeking loopt via `recordStat()` (2165), die `roundNumberFor()` gebruikt — het nummer van
de ronde die *nu bezig is*, vóór de ophoging. Dit gebeurt bij **elke** oefenvorm, niet alleen
bij Slim oefenen: ook een ronde "Alle woorden" of "Alle foute woorden" schrijft `g/f/s` bij
en hoogt aan het eind de rondeteller op.

De toaster-knop "Toch fout" roept `recordStat(item, false, toastPreState)` aan met de staat
van *vóór* het goede antwoord, zodat de correctie niet als "eerst goed, daarna fout" wordt
geboekt.

## 4. Het gewicht — `weightFor` (2484)

```js
return state.f > 0 ? 1 : 1 / intervalFor(state.g);
```

Dit is het aandeel van een ronde dat een woord gemiddeld opeist: een woord met interval 10
komt in 1 op de 10 rondes langs, dus gewicht 0,1. Een fout woord komt elke ronde, dus
gewicht 1.

De som van al die gewichten is precies de doorvoer die het schema zelf genereert — en die
som wordt gebruikt als gewenste rondegrootte. Dat is de elegantie van het ontwerp: de
rondegrootte is geen vast getal maar leest zichzelf af uit de planning. 50 woorden die
allemaal op interval 10 staan → `load = 5` → rondes van 5 woorden. Een deck vol verse
woorden → veel hoge gewichten → grotere rondes, tot het plafond.

## 5. De selectie — `selectSmartItems` (2521)

Eerst wordt in `buildSmartPlaylist` (2587) alles met `x` eruit gefilterd — bewust dáár en
niet in `selectSmartItems`, zodat uitgezette woorden ook niet meetellen voor het minimum
van 5.

De overgebleven woorden worden in vier pools verdeeld:

| pool | filter | sortering |
|---|---|---|
| `wrong` | `f > 0` | `-s` (recentst getoond eerst), dan `-f` (vaakst fout eerst) |
| `unseen` | geen staat in deze richting | **niet geschud** — deckvolgorde |
| `due` | `f === 0 && g > 0 && s <= round` | `g < 5` eerst, dan `s` oplopend (meest over tijd eerst) |
| `later` | `f === 0 && g > 0 && s > round` | `s` oplopend (eerstvolgende eerst) |

`wrong`, `unseen` en `known` (= `due` + `later`) dekken samen het hele deck, want elk
beantwoord woord heeft `f > 0` of `g > 0`. `sortShuffled` (2506) schudt eerst en sorteert dan
stabiel op de sleutels in volgorde van belang, zodat woorden die op álle sleutels gelijk zijn
in willekeurige volgorde staan.

### Waarom de foute woorden op recency sorteren

```js
const wrong = sortShuffled(
  entries.filter((e) => e.state.f > 0),
  (e) => -e.state.s,   // recentst getoond eerst
  (e) => -e.state.f,   // daarbinnen: vaakst fout eerst
);
```

`-s` is hier "recentst getoond eerst", want voor `f > 0` is `s` precies de laatste toonronde
plus één (§3). Het gevolg: de foutenstapel gedraagt zich als een **stapel**, niet als een
prioriteitswachtrij. Alle woorden die je deze ronde zag krijgen hetzelfde, hoogste `s`, dus de
volgende ronde staan ze weer vooraan. Zit de stapel voller dan `maxWrongNew`, dan blijf je
dezelfde groep drillen tot je hem kent; pas als die woorden goed beantwoord zijn verlaten ze de
pool en schuiven de oudere fouten naar voren. Dat is bewust: één groepje afmaken leert beter
dan elke ronde springen naar wat toevallig de hoogste fout-teller heeft.

De fout-teller is daarmee nog steeds de tweede sleutel — binnen de woorden van dezelfde
toonronde gaat het vaakst foute woord voor.

### Waarom de due-pool op `g` splitst

```js
const due = sortShuffled(
  known.filter((e) => e.state.s <= round),
  (e) => (e.state.g >= SMART_KNOWN_GOED ? 1 : 0),  // nog in opbouw eerst
  (e) => e.state.s,                                // daarbinnen: meest over tijd eerst
);
```

Zonder die eerste sleutel gaat een ronde in een volgroeid deck vooral op aan de berg goed
gekende woorden, terwijl de woorden die nog in opbouw zijn achteraan blijven hangen. Nu krijgen
die eerst de ruimte — dat is waar de vooruitgang zit, en dat is ook wat je in je statistieken
terugziet.

`SMART_KNOWN_GOED = 5` (2463) is geen willekeurig getal: `intervalFor(5) = 10`, het maximum
(§2). Vanaf `g = 5` staat een woord dus op het langste interval en is het "uitgeleerd" voor
zover het schema dat kan zien.

Een woord dat je fout had komt hier vanzelf in de voorste groep terecht: na een fout is
`g ≤ 2` (§3), dus zodra het de foutenpool verlaat staat het nog steeds vooraan in `due` tot het
weer ingesleten is.

De goed gekende woorden worden hier niet structureel door uitgehongerd. De voorste groep loopt
zichzelf leeg: elk goed antwoord verhoogt `g`, dus na vijf goede beurten verhuist een woord naar
de achterste groep. En een woord dat blijft falen heeft `f > 0` en zit dan in `wrong`, niet in
`due`. Alleen zolang je echt veel woorden in opbouw hebt, moeten de bekende woorden wachten —
en dat is precies de bedoeling.

`later` blijft puur op `s` sorteren: dat is de noodvoorraad voor als er te weinig due is, en
daar is "eerstvolgende ronde eerst" het eerlijkste.

### Het quotum voor onbekende woorden

```js
const wrongRoom = Math.min(wrong.length, maxWrongNew, Math.max(maxWrongNew - shaky, wrongLastRound));
const newRoom = Math.max(0, Math.min(unseen.length, maxWrongNew - wrongRoom - shaky));
```

Lees dit eerst met `shaky = 0` — de normale situatie. Dan staat er gewoon
`min(wrong.length, maxWrongNew)` en `min(unseen.length, maxWrongNew - wrongRoom)`, want
`max(maxWrongNew - 0, wrongLastRound)` wordt door de buitenste `min` toch op `maxWrongNew`
gekapt.

Alles wat je nog niet kent deelt één plafond (`maxWrongNew`, instelling "Max.
foute/nieuwe woorden", standaard **5**), verdeeld op volgorde van urgentie:
**eerst fout, dan nieuw**. Gevolg: zit je foutenstapel vol, dan komen er die ronde
géén nieuwe woorden bij. Eerst opruimen. Wie dat niet wil kiest "Alle nieuwe woorden".

Ging het de vorige ronde slecht, dan rekent dat plafond ook de woorden mee die je pas één keer
goed had: `shaky` is dan niet 0. Dat is de focus-rem hieronder.

### De focus-rem na een slechte ronde

```js
const wrongLastRound = wrong.filter((e) => e.state.s === round).length;
const shaky = wrongLastRound >= SMART_FOCUS_WRONG
  ? known.filter((e) => e.state.g <= SMART_FRESH_GOED).length
  : 0;
```

Had je de vorige ronde **3 of meer** woorden fout (`SMART_FOCUS_WRONG`), dan tellen ook de
**wankele** woorden mee voor het quotum: de woorden met `g = 1` (`SMART_FRESH_GOED`), die je
één keer goed had maar nog niet kent. Daarmee komen er geen onbekende woorden bij zolang je nog
met die stapel bezig bent — precies de focus waar het om gaat.

`s === round` is "vorige ronde fout": voor `f > 0` is `s` de laatste toonronde plus één (§3).
Dat zijn door de sortering ook meteen de eerste woorden in `wrong`.

Drie dingen om op te letten:

- **De fouten van de vorige ronde zijn de ondergrens** (`Math.max(…, wrongLastRound)`). Die
  komen hoe dan ook terug; de rem gaat ten koste van oudere fouten en van nieuwe woorden.
- **Alle** woorden met `g = 1` tellen mee, ook die nog niet due zijn. Dat is bewust: een woord
  dat je goed had staat op `s = ronde + 2`, dus wat je de vorige ronde voor het eerst goed had
  is deze ronde nog niet due. Zou de rem alleen naar `due` kijken, dan zou hij juist op het
  moment dat het nodig is niets doen.
- **Er is geen aparte tak voor "rem aan"**: met `shaky = 0` staan de quotumregels hierboven
  gelijk aan de oude formule. De `Math.max(0, …)` om `newRoom` is wel nodig, want
  `maxWrongNew - wrongRoom - shaky` kan negatief worden en `unseen.slice(0, -2)` zou dan
  stilletjes de láátste nieuwe woorden pakken in plaats van er nul te nemen.

De rem telt wat er klaarstaat, niet wat er uiteindelijk gekozen wordt: of die wankele woorden
deze ronde ook echt langskomen hangt af van `due`/`later` en de doelgrootte. Het is een rem, geen
reservering. En omdat `load` op het geknepen quotum is gebaseerd, wordt de ronde tijdens de rem
ook een paar woorden korter — ook dat past bij het doel: minder tegelijk.

### De doelgrootte

```js
const load = known.reduce((sum, e) => sum + weightFor(e.state), wrongRoom + newRoom);
const minItems = Math.min(SMART_MIN_ITEMS, cap, entries.length);
const target = Math.max(minItems, Math.min(cap, Math.round(load)));
```

- De onbekende woorden tellen mee met hun **gequoteerde** aantal, niet hun echte aantal —
  anders zou een vers lesbestand van 200 woorden de ronde laten exploderen.
- `known` telt mee met de gewichtensom over **alle** bekende woorden, ook die pas over 9
  rondes gepland staan.
- `SMART_MIN_ITEMS = 5` is de bodem, `cap` het plafond (`getMaxRoundSize()`, 1887): een
  getal, `Infinity` bij "Alle woorden", of gelijk aan `maxWrongNew` bij de stand
  "= Max. foute/nieuwe".

### Het vullen

```js
const chosen = wrong.slice(0, wrongRoom).concat(unseen.slice(0, newRoom));
for (const pool of [due, later]) {
  for (const entry of pool) {
    if (chosen.length >= target) break;
    chosen.push(entry);
  }
}
return chosen.slice(0, cap).map((e) => e.item);
```

Eerst de onbekende kern, dan aanvullen met woorden die aan de beurt zijn, en als die op zijn
met woorden die eigenlijk pas later gepland staan — dichtstbijzijnde ronde eerst. De
slotafkapping op `cap` is nodig omdat het quotum zelf al groter dan `cap` kan zijn.

### Volgorde binnen de ronde

De sortering in `selectSmartItems` bepaalt alleen *wat* gekozen wordt; de volgorde waarin je
het te zien krijgt staat daar los van. `buildSmartPlaylist` gooit er `ordered()` (1914)
overheen, en dat doet élke oefenvorm: willekeurig geschud, behalve als je "Originele volgorde"
hebt gekozen — dan zet `inDeckOrder()` (1899) de selectie terug in deckvolgorde, die de
lesbestanden in de volgorde van toevoegen achter elkaar plakt. De recency- en
fout-tellersortering zie je tijdens het oefenen dus nergens terug.

## 6. Een uitgewerkt voorbeeld

Deck van 60 woorden, `maxWrongNew = 5`, `cap = 10`, ronde 12. Stand: 3 fout, 20 nieuw,
37 bekend (allemaal `g ≥ 5`, dus interval 10).

- `wrongRoom = 3`, `newRoom = min(20, 5-3) = 2`
- `load = 3 + 2 + 37 × 0,1 = 8,7` → `target = max(5, min(10, 9)) = 9`
- Gekozen: 3 foute + de eerste 2 nieuwe in deckvolgorde + 4 uit `due` (of uit `later` als er
  minder dan 4 due zijn)

Die 37 bekende woorden staan allemaal op `g ≥ 5`, dus de 4 aanvullers komen uit één groep en
het is puur "meest over tijd eerst". Zaten er ook woorden met bijvoorbeeld `g = 2` tussen, dan
waren die eerst gekozen — ook als ze pas net over tijd zijn.

Waren er 8 foute woorden geweest in plaats van 3, dan hadden de 5 die je in ronde 11 zag
(`s = 12`) voorrang gekregen op de 3 die al sinds ronde 7 wachten (`s = 8`) — ook als die
laatste een hogere fout-teller hebben.

Stel nu dat die 3 foute woorden de fouten van ronde 11 zijn (`s = 12`) en dat er ook 2 woorden
met `g = 1` in het deck staan. Dan gaat de focus-rem aan: `shaky = 2`,
`wrongRoom = min(3, 5, max(5-2, 3)) = 3` en `newRoom = max(0, min(20, 5-3-2)) = 0`. Dezelfde 3
foute woorden, maar geen nieuwe erbij — en `load` zakt met 2, dus de ronde wordt ook iets
korter.

Beantwoord je die 3 foute allemaal goed, dan gaan ze naar `g = min(floor(g/2.5),2)+1`, worden
ze ingepland op ronde 12 + hun interval, en zakt `load` de volgende ronde — en komen er weer
meer nieuwe woorden binnen het quotum.

## 7. Vier eigenschappen om te weten

1. **De klok loopt op élke ronde.** Doe je drie rondes "Alle woorden" achter elkaar, dan
   staat de teller 3 verder en is in Slim oefenen ineens veel "due". Dat is consistent (je
   hebt die woorden ook echt gezien, en ze zijn ook echt geboekt), maar het is geen eigen
   klok per oefenvorm.
2. **`load` telt ook niet-due woorden mee**, en het vullen valt bewust terug op `later`. De
   ronde heeft dus altijd zijn doelgrootte, ook als er niets over tijd is — ten koste van het
   naar voren trekken van toekomstige herhalingen. Zo'n vroeg herhaald woord krijgt wel
   gewoon zijn volledige nieuwe interval mee, dus het schema loopt niet vast.
3. **`SMART_MIN_ITEMS = 5` en de `x`-filtering vóór de selectie** zorgen samen dat een
   bijna-uitgeleerd deck nog steeds rondes van 5 oplevert uit wat er nog over is, in plaats
   van rondes van 1.
4. **De focus-rem gaat vanzelf weer uit.** Hij kijkt alleen naar de vórige ronde, dus één ronde
   met minder dan 3 fouten zet hem uit. En een wankel woord telt niet meer mee zodra je het een
   tweede keer goed hebt: dan is `g = 2`.

## Bijlage: waar staat wat

| Onderdeel | Regels in `index.html` |
|---|---|
| `SMART_MIN_ITEMS` / `SMART_MAX_INTERVAL` / `SMART_KNOWN_GOED` | 2460 / 2461 / 2463 |
| `SMART_FOCUS_WRONG` / `SMART_FRESH_GOED` | 2466 / 2467 |
| `fibonacci` / `intervalFor` / `weightFor` | 2469 / 2478 / 2484 |
| `nextState` | 2488 |
| `sortShuffled` | 2506 |
| `selectSmartItems` | 2521–2583 |
| `buildSmartPlaylist` / `startSmartRound` | 2587 / 2600 |
| Per-woord staat + `roundNumber` | 1180–1184 |
| `stateFor` / `recordStat` | 2151 / 2165 |
| Rondeteller ophogen (`showEndScreen`) | 2348 |
| `ordered` / `inDeckOrder` / `wrongItems` | 1914 / 1899 / 1924 |
| `getMaxWrongNew` / `getMaxRoundSize` | 1879 / 1887 |
| `PRACTICE_OPTIONS` | 2711 |
| `persistState` / `loadStoredState` / storage key | 1267 / 1250 / 1196 |
