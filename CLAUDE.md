# Akkuvahti

PWA (yksi `index.html`), joka seuraa sähköauton akun kuntoa automaattisesti
ajoista. Data tulee Polestar-palvelusta (`C:\ohjelmat\Polestar-palvelu`,
`GET /v1/ajot`), joka tunnistaa ajot ja hakee ulkolämpötilan. Sovellus ei
kirjoita palveluun mitään.

Lue ennen muutoksia: `docs/SUUNNITELMA.md` ja palvelun `docs/API.md`.

## Pysyvät linjaukset (käyttäjän päätökset 11.11.2026)

* Vain ajot, joissa varaus laskee vähintään 20 %-yksikköä (`RAJA`).
* Ei latauksia: kWh:ta ei saada automaattisesti. Lataukset arvioidaan
  vanhalla Akun kapasiteetti -sovelluksella, jota **ei muuteta**.
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
