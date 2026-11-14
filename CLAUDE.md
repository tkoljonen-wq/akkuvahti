# Akkuvahti

PWA (yksi `index.html`), joka seuraa sähköauton akun kuntoa automaattisesti
ajoista. Data tulee Polestar-palvelusta (`C:\ohjelmat\Polestar-palvelu`,
`GET /v1/ajot`), joka tunnistaa ajot ja hakee ulkolämpötilan. Sovellus ei
kirjoita palveluun mitään.

Lue ennen muutoksia: `docs/SUUNNITELMA.md`, `docs/LAMMITYSKORJAUS.md` ja palvelun `docs/API.md`.

## Pysyvät linjaukset (käyttäjän päätökset 11.11.2026)

* Vain ajot, joissa varaus laskee vähintään 20 %-yksikköä (`RAJA`). Lyhyempiä
  ei haeta eikä näytetä; ne eivät ole missään laskennassa (vahvistettu 14.11.2026).
* Lataukset (lisätty 11.11.2026): lista tulee palvelusta (`/v1/lataukset?tapa=kytkenta`,
  AC/DC autolta), kWh syötetään käsin halutessa. Laskennassa vain lataukset,
  joilla on kWh ja varauksen nousu ≥ 20. kWh:t, kapasiteetti uutena ja
  poissulut ovat vain localStoragessa → varmuuskopio JSON-tiedostona.
* Lämmityskorjaus (lisätty 14.11.2026): kylmän lähdön lämmityslisä vähennetään
  varauksen laskusta; alustava kaava on käytössä, kunnes kerroin on opittu
  (≥ 4 kylmää ja ≥ 4 lämmintä lähtöä). Pois-kytkin asetuksissa
  (`akkuvahti-prefs-v1`, `heat`). Palvelu antaa `tauko_ennen_s`, `lataus_ennen`,
  `esilammitys`, `tauko_talli` ja `kwh_auto`. Hallista lähteneet ajot
  (`tauko_talli` tai käsin `data.talli`) eivät ole mukana korjauksessa eivätkä
  oppimisessa. Perustelut: `docs/LAMMITYSKORJAUS.md`.
* Vanhaa Akun kapasiteetti -sovellusta **ei muuteta**.
* Lämpötila mukana (Open-Meteo palvelun kautta). Sovellus ei käsittele
  sijainteja eikä avaimella ole sijaintioikeutta.
* Käyttöliittymä suomeksi, optimoitu Android-puhelimelle.

## PWA ja jaettu origin

* GitHub Pages, origin `https://tkoljonen-wq.github.io` on yhteinen muiden
  sovellusten kanssa. Välimuistin etuliite `akkuvahti-`, localStorage-avaimet
  `akkuvahti-avain`, `akkuvahti-url`, `akkuvahti-data-v1`, `akkuvahti-prefs-v1`.
  Älä koske muiden sovellusten tietoihin.
* `manifest.json` id `akkuvahti`, PNG-ikonit `icons/` (maskable ~72 % turva-alue).
* `sw.js` network first, `cache: 'no-cache'`, rekisteröinti `updateViaCache: 'none'`.

## Testaus paikallisesti

Palvelu paikallisena (projektin `Polestar-palvelu` juuresta), PWA esim.
`python -m http.server 8791` tässä kansiossa:

    npx wrangler dev --local --var SALLITUT_ORIGINIT:http://localhost:8791

Palvelun osoitteen voi vaihtaa Asetukset → Palvelun osoite (tallentuu
`akkuvahti-url`). Varmista testin jälkeen, ettei `wrangler`/`workerd` jää päälle.
Tällä koneella `.claude/launch.json` (gitignore) käynnistää molemmat (`pwa`,
`palvelu`); paikalliseen D1:een saa oikean datan viemällä taulut `--remote`-kannasta.
