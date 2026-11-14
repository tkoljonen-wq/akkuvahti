# Lämmityskorjaus — suunnitelma (14.11.2026)

Kylmän lähdön (pitkä tauko kylmässä) ylimääräinen kulutus akun ja ohjaamon
lämmitykseen huomioidaan ajojen laskennassa kaavalla, jonka kerroin opitaan
ajoista sitä mukaa kuin pitkän ja lyhyen tauon jälkeisiä ajoja kertyy.
Tila: **toteutettu ja julkaistu 14.11.2026** päätöskohtien 1–4 suositusten mukaan
(luku 7), toteutus luvussa 8, lämmin halli luvussa 9.

## 1. Havainto 14.11.2026 (D1, Lahti ↔ pääkaupunkiseutu)

| | Meno 9.39–10.59 | Paluu 11.58–13.15 |
|---|---|---|
| Matka | 112,8 km | 114,4 km |
| Varaus | 100 → 56 % (44 %-yks) | 56 → 17 % (39 %-yks) |
| Toimintamatka 100 %:lla | **256 km** | **293 km** |
| Keskinopeus | 89 km/h (`auto`) | 93 km/h (`erotus`) |
| Ulkolämpötila (Open-Meteo) | 0,7 °C | 1,0 °C |
| Tauko ennen lähtöä | **12,1 h** (edellinen ajo 13.11. klo 21.32) | **1,0 h** |
| Lataus ennen | AC 60 → 100 %, päättyi 6.30, johto irti 9.38 | – |
| Auton kulutusmittari¹ | 29,0 kWh = 25,7 kWh/100 km | 26,1 kWh = 22,8 kWh/100 km |
| Kapasiteetti mittarista (kWh ÷ Δvaraus) | 66,0 kWh | 67,0 kWh |

¹ Akkunäytteen `kulutus_kwh100` on käsin nollattavan osamatkan keskikulutus:
kWh = kulutus × `osamatka_kasi_km` ÷ 100, erotus ajon alun ja lopun välillä.
Tarkkuus ±0,5 kWh (kulutus pyöristetty 0,1:een, osamatka 380–610 km).

Johtopäätökset:

* Meno kulutti **5 %-yks ≈ 2,9 kWh (±0,7) enemmän** lähes samalla reitillä,
  samassa lämpötilassa ja hieman pienemmällä nopeudella. Meno on lisäksi
  kokonaisuutena alamäkeä (Lahti ≈ 90 m → rannikko), mikä tekisi erosta
  vielä hieman suuremman.
* Auton oma mittari antaa molemmista ajoista saman kapasiteetin (66–67 kWh),
  joten ylimääräinen varauksen lasku on **oikeaa energiaa**, ei varausasteikon
  epälineaarisuutta (100 %:n yläpää).
* Suuruusluokka täsmää fysiikkaan: akkupaketti ~400 kg × ~1 kJ/(kg·K) × 15 K
  ≈ 1,7 kWh, ohjaamon lämmitys 0,5–1 kWh, kylmä voimansiirto ja renkaat loput.
* Sama suunta 11.11.: meno klo 11.16 (tauko 0,8 h, kaksi lyhyttä ajoa ennen)
  3,24 km/%-yks, paluu klo 16.56 (tauko 3,2 h) 2,89 km/%-yks eli +2,4 kWh/100 km.
  Paluussa tosin myös pimeä ja ruuhka, ja mittari oli nollattu kesken ajon.

**Vaikutus malliin:** 20 %-yksikön ajoilla (≥ 50 km) kylmä lähtö heittää
toimintamatkaa 5–15 %. Se on suurempi kuin kuluminen, jota yritetään
mitata (pari % vuodessa), joten korjaus kannattaa. Toisaalta "identtisten"
ajojen hajonta on samaa luokkaa kuin itse ilmiö, joten opittu kerroin on
otettava käyttöön vasta, kun se erottuu kohinasta (sama periaate kuin
lämpötila- ja nopeuskorjauksilla).

## 2. Mitä palvelu voi kertoa jokaisesta ajosta

Kaikki lasketaan nykyisistä näytteistä `src/ajot.js`:ssä, PWA ei tarvitse
uusia oikeuksia:

| Kenttä (`/v1/ajot`) | Lähde | Huomio |
|---|---|---|
| `tauko_ennen_s` | ajon alku − edellisen käytön loppu (edellinen km-näyte ennen alkunäytettä) | null jakson ensimmäiselle ajolle, jos edellistä ei ole |
| `lataus_ennen` `{tyyppi, loppu}` | viimeinen lataava akkunäyte tauon aikana | DC lämmittää akun → lasketaan käytöksi (tauko alkaa DC-latauksen lopusta); AC ei |
| `kwh_auto`, `kwh_auto_virhe` | auton kulutusmittari (yllä ¹) | null, jos osamatka nollattiin ajon aikana tai näyte puuttuu |
| `tauko_talli` | tauon paikka (viimeinen sijainti ennen lähtöä) osuu palveluun asetettuun lämpimään paikkaan (taulu `lammin_paikka`) | Hallin lämpötila vaihtelee ulkoilman mukaan, joten ajo jätetään **kokonaan pois** korjauksesta ja oppimisesta (käyttäjän päätös 14.11.2026). Sovelluksessa Halli-painike korjaa tunnistuksen käsin |
| `esilammitys` (valinnainen) | uusi keruualue `telemetry/parking-climatization` (`runningStatus`, `startedAt`, `endingAt`, `startReason`) | +288 Polestar-kutsua/vrk (kiintiö 10 000). Esilämmitetty auto on "lämmin lähtö" pitkästä tauosta huolimatta. Autossa on ajastin (to 4.20, ei aktiivinen) |

Auton automaattisen osamatkan nollautuminen (`nopeus_lahde = auto`) kertoo
sekin ~4 h seisonnasta, mutta tauko lasketaan suoraan näytteistä.

## 3. Malli

Nykyinen: toimintamatka R = matka ÷ Δvaraus × 100, painotettu regressio
(km-lukema, lämpötilan polvi +15 °C, nopeus 70 km/h).

Ehdotus: varauksen laskuun lisätään vakiotermi, joka ei riipu matkasta:

```
Δvaraus = matka ÷ R(km, T, v) × 100 + H
H = h × q                                   lämmityslisä, %-yksikköä
q = max(0, T_H − T) × (1 − exp(−tauko / τ))  lämmitystarve
```

* `T_H = +10 °C`: tämän yläpuolella lisää ei ole. `τ = 3 h`: tunnin tauon
  jälkeen lisästä on jäljellä 28 %, 12 h jälkeen 98 %. (Ohjaamo jäähtyy
  tunneissa, akku hitaammin; yksi aikavakio riittää, kunnes dataa on
  erottamaan ne.) Molemmat ovat vakioita, joita ei opita.
* `h` (%-yks per °C-yksikköä) on **opittava kerroin**. Alkuarvo fysiikasta
  `h₀ = 0,35` (0 °C, pitkä tauko → 3,5 %-yks ≈ 2,3 kWh; tämän päivän
  havainto 5 ± 2 %-yks on sen kanssa sopusoinnussa). Kerroin ei voi olla
  negatiivinen, ja H rajataan enintään 25 %:iin ajon Δvarauksesta.
* Vakioitu ajo: `Δvaraus_korj = Δvaraus − H`, `R_korj = matka ÷ Δvaraus_korj × 100`.
  Muu malli jatkaa ennallaan korjatulla arvolla. Toimintamatka-otsikko
  tarkoittaa jatkossa lämpimän lähdön toimintamatkaa. Samankaltaisten ajojen
  parivertailu käyttää samaa korjattua arvoa.
* Ajon hyväksyntä (`RAJA` 20 %-yks) tehdään edelleen korjaamattomasta
  Δvarauksesta.

### Oppiminen

Kerroin päivitetään joka laskennalla kaikista hyväksytyistä ajoista:

1. Sovitetaan nykyinen malli korjatulla Δvarauksella (nykyinen `h`).
2. Jäännökset varausyksiköissä `e = Δvaraus − matka ÷ R̂ × 100`, josta
   painotettu sovitus origon kautta `h = Σ w·e·q ÷ Σ w·q²` ja keskivirhe
   (paino = Δvaraus kuten nyt).
3. Toistetaan 1–2 kolme kertaa (R̂ riippuu h:sta vain heikosti).

Tilat kuten lämpötila- ja nopeuskorjauksilla:

| Tila | Ehto | Käytössä |
|---|---|---|
| `data` | kylmiä lähtöjä (q ≥ 5) alle 4 tai lämpimiä (q ≤ 2) alle 4 | alkuarvo `h₀`, näytetään "alustava kaava" |
| `on` | kumpiakin ≥ 4, `h ≥ 2 × keskivirhe`, `0 < h ≤ 1,0` | opittu `h` |
| `unsure` | dataa riittää, mutta vaikutus ei erotu | ei korjausta (alkuarvokin pois) |

Vaihtoehto, joka hylättiin: kaksi luokkaa (kylmä/lämmin lähtö) ja niiden
keskiarvojen ero. Yksinkertaisempi, mutta ei osaa käyttää tunnin taukoa
osittain eikä yleistää lämpötilan yli.

### Mitä korjaus tekisi tämän päivän ajoille (alkuarvolla)

| | q | H | Δvaraus_korj | Toimintamatka |
|---|---|---|---|---|
| Meno (0,7 °C, 12,1 h) | 9,1 | 3,2 %-yks | 40,8 | 256 → **276 km** |
| Paluu (1,0 °C, 1,0 h) | 2,6 | 0,9 %-yks | 38,1 | 293 → **300 km** |

Ero kapenee 37 km:stä 24 km:iin. Opittu kerroin olisi tästä parista
~0,8, mutta yksi pari ei riitä (ks. kohina yllä).

## 4. Muutokset

### Polestar-palvelu (`C:\ohjelmat\Polestar-palvelu`)

* `src/ajot.js`: `tauko_ennen_s`, `lataus_ennen`, `kwh_auto` (+ virhe).
  `api.js` hakee kaksi km-näytettä ennen jaksoa (nyt yksi), jotta jakson
  ensimmäiselläkin ajolla on tauko. Testit `test/ajot.test.js`, `docs/API.md`.
* Valinnainen: keruualue `parking-climatization` → taulu `esilammitys`,
  kenttä `esilammitys` ajoon (päättyi ≤ 30 min ennen lähtöä). `schema.sql`,
  `keruu.js`, `wrangler d1 execute --remote`.
* Hakemisto **ei ole git-repo** → `git init` ennen muutoksia.
* Julkaisu `npx wrangler deploy`; vanha PWA toimii uusien kenttien kanssa.

### Akkuvahti (`index.html`)

* Haku: uudet kentät talteen; jos vanhasta datasta puuttuu `tauko`,
  haetaan kaikki uudelleen (`haettu = null`, kuten latausten lisäyksessä).
* Malli: vakiot `T_H`, `TAU_H`, `H_PRIOR`, `H_MIN_N`; `q(d)`, `h`:n sovitus
  ja tilat `driveModel`iin; `analyzeDrives` ja `similarDrives` käyttävät
  korjattua Δvarausta.
* Kunto-kortti: uusi tunnusluku "Lämmityslisä kylmän lähdön jälkeen" —
  "kerätään dataa (kylmiä 2/4, lämpimiä 2/4) · alustava kaava 3,5 %-yks
  0 °C:ssa" / "opittu 0,5 ± 0,1 %-yks/°C" / "ei selvää vaikutusta".
  kWh-vastine kapasiteetista (latausmalli; varalla auton mittarin mediaani).
* Ajot-lista: "tauko 12 h · DC 2 h sitten · lämmitys −3,2 %-yks" ja
  auton mittarin kWh, kun se on käytettävissä.
* Asetukset: "Lämmityskorjaus: automaattinen / pois".
* `docs/SUUNNITELMA.md` päätöstaulukko ja tila, `CLAUDE.md` linjaukset.

### Tarkistus

* Toisto nykyisellä datalla (skripti, joka ajaa `tunnistaAjot` D1-viennille):
  tila `data`, alkuarvo tuottaa taulukon 3 luvut.
* Paikallinen testi `wrangler dev --local` + `http.server 8791` (CLAUDE.md).

## 5. Datan tarve

Nyt 20 %-yksikön ajoja on 4 (11.11. ×2, 14.11. ×2). Opittu kerroin vaatii
≥ 4 kylmää ja ≥ 4 lämmintä lähtöä alle +10 °C:ssa, eli nykyisellä
ajotahdilla arviolta 3–6 viikkoa. Lämpötilariippuvuuden muoto (lineaarinen
`T_H`:n alapuolella) on oletus; data kiinnittää kertoimen lähinnä 0 °C:n
tienoilla, missä sillä on eniten merkitystä.

## 6. Sivulöydös: auton kulutusmittari

Käsin nollattavan osamatkan keskikulutuksesta saa jokaiselle pitkälle
ajolle kWh:t ±0,5 kWh ja siten **kapasiteetin ilman käsin syötettyä
latauskWh:ta** (tänään 66,0 ja 67,0 kWh; 11.11. paluu 64,8 kWh nollauksen
jälkeen). Mittari laskee akusta otetun energian, joten se on eri asteikko
kuin laturin kWh (ei latauksen häviöitä). Ei tähän muutokseen — jatkoidea
Lataukset-mallin rinnalle.

## 7. Päätettävää

1. **Alustava kaava heti käyttöön** (suositus: kyllä, merkitään alustavaksi,
   pois-kytkin asetuksissa) vai korjaus vasta, kun kerroin on opittu?
2. **Esilämmityksen keruu** palveluun (suositus: kyllä, jos esilämmitystä
   käytetään talvella; muuten myöhemmin).
3. **Auton mittarin kWh** mukaan `/v1/ajot`-vastaukseen ja ajolistaan
   (suositus: kyllä, pieni lisä samassa muutoksessa).
4. `T_H = +10 °C` ja `τ = 3 h` vakioina (suositus: kyllä; opitaan vain `h`).

## 8. Toteutus 14.11.2026

* Polestar-palvelu (git alustettu 14.11.2026): `src/ajot.js` antaa
  `tauko_ennen_s`, `lataus_ennen`, `esilammitys`, `kwh_auto` ja
  `kwh_auto_virhe` (`mittariKwh`); `api.js` hakee kaksi km-näytettä ennen
  jaksoa ja esilämmitysnäytteet; keruualue `telemetry/parking-climatization`
  → taulu `esilammitys` (`schema.sql`, ajettu D1:een). Testit `npm test`
  (14 ok). Julkaistu `wrangler deploy` 14.11.2026.
* Akkuvahti: vakiot `T_H`, `TAU_H`, `H_PRIOR`, `H_MIN_N`, `H_COLD`, `H_WARM`,
  `H_MAX`, `H_SHARE`; `heatQ`, `heatH`, `learnH`, `analyzeDrives` (kolme
  iteraatiota), `rangeOf` korjatulla Δvarauksella ja `rangeRaw` kuvaajan
  mitatuille pisteille; Kunto-kortin tunnusluku, ajolistan rivi (tauko,
  lataus, lämmitys, mittarin kWh), Asetukset → Lämmityskorjaus (`prefs.heat`).
  Vanha data ilman `tauko`-kenttää haetaan kerran uudelleen.
* Paikallinen testi 14.11.2026 (D1-vienti paikalliseen kantaan, `wrangler dev`,
  `http.server 8791`): tila `data`, tunnusluku 3,5 %-yks ≈ 2,3 kWh, kylmiä
  lähtöjä 1 / lämpimiä 1; toimintamatkat 328 / 297 / 276 / 300 km kuten
  luvussa 3, viimeisten ajojen toimintamatka 301 km (pois-kytkimellä 291 km).

## 9. Lämmin halli 14.11.2026

Auto on maanantaisin ja tiistaisin työpäivän ajan lämpimässä parkkihallissa,
jonka lämpötila vaihtelee ulkoilman mukaan. Polestarin rajapinnasta ei saa
ulkolämpötilaa (ainoa lämpötila on pysäköintilämmityksen ohjaamolämpötila,
jota auto ei testissä lähettänyt). Ratkaisu:

* Palvelun taulu `lammin_paikka` (nimi, lat, lon, sade_m). Oletuspaikka
  "Työ" 60,983747 / 25,660238, säde 150 m (käyttäjän antama; datasta
  maanantain 9.11. pysäköinti klo 7.55–16.29 oli 70 m:n päässä). Tauon
  paikka = viimeinen sijainti ennen lähtöä → `/v1/ajot`-kenttä `tauko_talli`.
  Koordinaatit pysyvät palvelussa, sovellus saa vain lipun.
* Sovellus: hallista lähtenyt ajo (`talli` palvelusta tai käsin merkitty
  `data.talli[id]`, mukana varmuuskopiossa) saa lämmitystarpeen NaN → kun
  korjaus on käytössä, ajo ei ole mukana mallissa eikä kertoimen oppimisessa.
  Ajolistan Halli-painike vaihtaa merkinnän; "merkitty" näkyy, kun se
  poikkeaa palvelun tunnistuksesta. Ilman korjausta (asetus pois) ajot ovat
  mukana normaalisti.
* Lisäys/muutos: `wrangler d1 execute polestar --remote --command "INSERT INTO
  lammin_paikka (nimi, lat, lon, sade_m, lisatty) VALUES (...)"`. Tiistain
  10.11. pysäköintipaikka 60,974 / 25,677 (1,4 km työpaikasta) ei ole listalla.
* Lyhyet ajot (alle 20 %-yks) eivät näy sovelluksessa eivätkä ole mukana
  missään laskennassa, joten niitä ei tarvitse merkitä (käyttäjän päätös
  14.11.2026). Halli-painike on vain laskennassa olevia pitkiä ajoja varten,
  joiden lähtöpaikkaa palvelu ei tunnista oikein.
* Odotettavissa: kerroin alkaa oppia, kun kylmiä ja lämpimiä lähtöjä on
  kumpiakin vähintään 4 (14.11.2026: 1 / 1).
