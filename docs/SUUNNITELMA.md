# Akkuvahti — suunnitelma ja toteutus

PWA akun kunnon seurantaan: ajot automaattisesti, lataukset automaattisesti
listattuina ja kWh käsin. Osoite https://tkoljonen-wq.github.io/akkuvahti/.
Vanha *Akun kapasiteetti* (`C:\ohjelmat\Akun kapasiteetti`) jää ennalleen.

## Päätökset

| Päivä | Päätös |
|---|---|
| 11.11.2026 | Uusi ohjelma *Akkuvahti*, vanhaa ei muokata. |
| 11.11.2026 | Vain ajot, joissa varaus laskee vähintään 20 %-yksikköä. Muut jätetään pois. |
| 11.11.2026 | Lataukset listataan automaattisesti, kWh syötetään käsin halutessa. Laskennassa vain lataukset, joilla on kWh. |
| 11.11.2026 | Ulkolämpötila mukana (Open-Meteo), auton sijainnista ja ajon ajalta. |
| 11.11.2026 | Ajojen tunnistus ja lämpötila Polestar-palvelussa (`/v1/ajot`), PWA vain lukee. |

## Arkkitehtuuri

```
Polestar API ──► Polestar-palvelu (Cloudflare Worker, D1 EU)
                   cron 5 min: km, akku, sijainti, saatavuus
                   GET /v1/ajot: ajot näytteistä + lämpötila (välimuisti ajo_lampo)
                   ▼  avain: km + akku (ei sijaintia)
                 Akkuvahti (PWA, GitHub Pages): malli ja kuvaajat selaimessa
```

Miksi palvelussa: PWA ei tarvitse sijaintioikeutta, lämpötila haetaan
kerran ja tallennetaan, ja Ajopäiväkirja voi käyttää samaa tunnistusta.

## Ajojen tunnistus (`Polestar-palvelu/src/ajot.js`)

**Havainto 11.11.2026 (D1, 8.–11.11.):** auto lähettää km-näytteen käytön
alkaessa (lukema ennallaan, saatavuus `DRIVING`) ja päättyessä (lukema
kasvanut) omalla sekunnin tarkalla aikaleimallaan. Ajon aikana km- ja
sijaintinäytteitä ei tule; akkunäytteitä tulee noin 5 min välein.

* Ajo = peräkkäiset km-näytteet, joiden välillä lukema kasvaa ≥ 0,3 km
  (pysäköinnin jälkeen lukema tarkentuu ~0,1 km, se ei ole ajo).
* Alku = ensimmäinen `DRIVING` edellisen näytteen jälkeen.
* Alle 15 min tauot ilman latausta yhdistetään samaksi ajoksi.
* Varaus alussa = viimeisin akkunäyte ennen alkua. Lopussa = viimeinen
  ei-lataava näyte (tai seuraava, jos se on 15 min sisällä eikä lataa).
  Esim. 11.11. ajo päättyi 42 %:iin ja pikalataus alkoi 2 min myöhemmin.
* Jos ajon aikana on latausnäyte, ajo merkitään `latausta` eikä ole mukana.

### Keskinopeus

Polestar antaa kaksi keskinopeutta, kumpikin osamatkan alusta laskettuna:

* `averageSpeedKmPerHour` — käsin nollattava osamatka (satoja km). Ei ajokohtainen.
* `averageSpeedKmPerHourAutomatic` (vain `raaka`-JSONissa) — automaattinen
  osamatka, joka nollautuu, kun auto on seissyt noin 4 h.

Ajon keskinopeus valitaan järjestyksessä:

1. **`auto`:** automaattinen osamatka alkoi tästä ajosta → auton oma luku
   (sama, jonka vanha sovellus sai käsin auton näytöstä).
2. **`erotus`:** ajoaika = matka₂/nopeus₂ − matka₁/nopeus₁. Hyväksytään vain,
   jos ±0,5 km/h:n pyöristys muuttaa tulosta enintään 6 km/h.
3. **`aika`:** matka ÷ (loppu − alku). 5–8 % auton lukua pienempi, koska
   aikaan sisältyy käynnistyksen ja liikkeellelähdön väli. Merkitään ≈.

| Ajo (2026) | km | aika-arvio | auto / erotus |
|---|---|---|---|
| 9.11. 7:36 | 11,0 | 34 km/h | 37 (`auto`) |
| 9.11. 16:29 | 10,6 | 40 km/h | 42 (`auto`) |
| 9.11. 18:15 | 9,5 | 41 km/h | 44 (`erotus`) |
| 11.11. 11:16 | 158,8 | 85 km/h | 88 (`erotus`) |
| 11.11. 16:56 | 173,6 | 82 km/h | 85 (`erotus`) |

### Lämpötila (`Polestar-palvelu/src/saa.js`)

Open-Meteon tuntilämpötiloista keskiarvo ajon ajalta (10 min välein
interpoloituna) alku- ja loppupaikassa, paikkojen keskiarvo. Koordinaatit
pyöristetään 2 desimaaliin. Yli 80 pv vanhat arkistorajapinnasta.
Tallennetaan vain lämpötila tauluun `ajo_lampo` (tunniste = alku-loppu).

## Lataukset

* Palvelu: `/v1/lataukset?tapa=kytkenta` (`tunnistaKytkennat`, `src/johdetut.js`).
  Yksi lataus = kaapeli kiinni → irti, koska kotilaturin ja latausaseman
  kWh-lukema on koko kytkennältä (ajastetun latauksen tauot mukana).
  AC/DC tulee autolta (`chargingType`). Varaus ennen = viimeisin näyte ennen
  kytkentää, jälkeen = suurin varaus kytkennän aikana. Lämpötila 6 h
  keskiarvo ennen latauksen loppua (kuten vanhassa sovelluksessa).
* Sovellus: Lataukset-välilehti, kWh-kenttä jokaisella rivillä (pilkku tai
  piste). Malli vanhasta sovelluksesta: kapasiteetti = kWh ÷ Δvaraus × 100,
  painotettu regressio km:n mukaan, AC/DC-vakiointi kun kumpaakin ≥ 3,
  lämpötilakorjaus kun dataa riittää. SOH = kapasiteetti ÷ kapasiteetti uutena
  (asetus). Varmuuskopio JSON (kWh, kapasiteetti, poissulut; ei avainta).
* Havainto 11.11.: kolme kytkentää — AC 73→100 % (10.–11.11. yö), DC 42→67 %,
  AC 6→18 % (irrotettu kesken).

## Sovellus

* **Kunto:** toimintamatka 100 %:n varauksella, suhde ensimmäisiin ajoihin,
  trendi / 10 000 km, lämpötilan ja nopeuden vaikutus, kuvaaja (km tai pvm),
  samankaltaisten ajojen vertailu, tiedonkeruun tila.
* **Ajot:** lista; napautus jättää ajon pois laskennasta (tallentuu puhelimeen).
* **Asetukset:** avain, palvelun osoite, laskentatapa, tietosuoja, tietojen poisto.

Malli on siirretty vanhasta sovelluksesta sellaisenaan: painotettu
regressio (paino = varauksen lasku), lämpötilan polvi +15 °C,
nopeusvakiointi 70 km/h, korjaukset vain kun data riittää.

Haku: ensimmäisellä kerralla 8.11.2026 alkaen 62 pv:n jaksoissa, sen jälkeen
viimeiset 3 pv ja ajot, joilta lämpötila puuttuu (60 pv). Ajot tallennetaan
localStorageen (`akkuvahti-data-v1`).

## Tila

| # | Vaihe | Tila |
|---|---|---|
| 1 | Palvelu: `/v1/ajot`, `ajo_lampo`, testit | ✅ 11.11.2026, testattu paikallisesti oikealla datalla |
| 2 | Akkuvahti-PWA | ✅ 11.11.2026, testattu paikallisesti |
| 3 | Julkaisu: `ajo_lampo` D1:een, `wrangler deploy`, GitHub Pages | ✅ 11.11.2026, https://tkoljonen-wq.github.io/akkuvahti/ (repo `tkoljonen-wq/akkuvahti`) |
| 4 | Avain sovellukseen: `akun-kunto` (km + akku) | ✅ 11.11.2026 |
| 5 | Lataukset ja kWh-syöttö, julkaistu ja todettu toimivaksi | ✅ 11.11.2026 |

Jatkoideoita: vanhan sovelluksen ajojen tuonti (trendi heti pidemmältä ajalta).
