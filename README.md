# Deploy automat GitHub → cPanel (FTPS)

Publică automat un site **React + Vite** în `public_html` la fiecare `git push` pe ramura `main`, folosind **GitHub Actions**. Nu ai nevoie de SSH, cron jobs sau Git instalat pe hosting.

> **Important:** Acest repository conține un *model de workflow*. Fișierul `.github/workflows/deploy.yml` trebuie copiat în **repository-ul fiecărui site** pe care vrei să îl publici.

## 1. Află IP-ul real al serverului din cPanel

1. Autentifică-te în **cPanel**.
2. Intră în **Server Information** (Informații server).
3. Caută **Shared IP Address** sau **Dedicated IP Address**.
4. Copiază IP-ul afișat. Îl vei folosi pentru conectarea FTP, în secretul `FTP_SERVER` din GitHub.

**De ce IP-ul și nu domeniul?** Dacă domeniul folosește Cloudflare cu proxy activ (nor portocaliu), conexiunea FTP pe portul 21 poate expira cu `ETIMEDOUT`. IP-ul de origine evită această problemă. **Atenție:** dacă certificatul FTPS nu acoperă IP-ul, conexiunea poate eșua la verificarea certificatului. În acest caz, folosește un hostname FTP **DNS only** (nor gri), care indică spre IP-ul real al serverului. Nu dezactiva verificarea certificatului.

## 2. Creează utilizatorul FTP în cPanel

1. Intră în **cPanel → FTP Accounts**.
2. Creează un cont FTP dedicat deploy-ului (de exemplu `github@domeniul-tau.ro`) și setează o parolă puternică.
3. La câmpul **Directory**, alege directorul site-ului, **`public_html`** (sau calea completă indicată de cPanel). Pentru un domeniu adițional, folosește directorul-rădăcină al acelui site, nu automat `public_html`.
4. Verifică în **FTP Accounts** că utilizatorul este restricționat la directorul corect.
5. La **Configure FTP Client**, confirmă că serverul acceptă **FTP / explicit FTPS pe portul 21**.

Dacă utilizatorul FTP începe direct în `public_html`, atunci în workflow setezi **`server-dir: /`**. `/` reprezintă rădăcina *contului FTP*, nu rădăcina întregului server.

## 3. Copiază workflow-ul în proiect

Copiază fișierul din acest template:

```text
.github/workflows/deploy.yml
```

în aceeași cale din repository-ul **site-ului** tău. Workflow-ul existent este pregătit pentru React + Vite și publică **conținutul** folderului `dist/`, nu folderul `dist` în sine.

## 4. Configurează GitHub Secrets

În repository-ul site-ului, mergi la **Settings → Secrets and variables → Actions → New repository secret** și adaugă:

| Nume | Ce introduci |
| --- | --- |
| `FTP_SERVER` | IP-ul real din **cPanel → Server Information** sau un hostname FTP DNS-only valid pentru certificatul FTPS |
| `FTP_USERNAME` | Utilizatorul FTP creat la pasul 2, de exemplu `github@domeniul-tau.ro` |
| `FTP_PASSWORD` | Parola utilizatorului FTP |

Nu introduce aceste date direct în fișierul YAML și nu le publica pe GitHub.

## 5. Verifică setările de publicare

În `.github/workflows/deploy.yml`:

```yaml
with:
  server: ${{ secrets.FTP_SERVER }}
  username: ${{ secrets.FTP_USERNAME }}
  password: ${{ secrets.FTP_PASSWORD }}
  protocol: ftps
  port: 21
  local-dir: ./dist/
  server-dir: /
```

- `local-dir: ./dist/` — pentru React + Vite; presupune că `npm run build` generează `dist/`.
- `server-dir: /` — când utilizatorul FTP este restricționat direct la `public_html`.
- **Nu folosi** `server-dir: /public_html/` dacă utilizatorul FTP intră deja în acel folder: ai putea publica într-un subfolder greșit.

Proiectul React + Vite trebuie să aibă `package.json`, `package-lock.json` și un script `build` funcțional.

## 6. Testează deploy-ul

Fă mai întâi un **backup al folderului `public_html`**. Sincronizarea FTP poate modifica sau șterge fișiere existente. Nu ține în directorul sincronizat fișiere încărcate de utilizatori, date sau alte fișiere pe care vrei să le păstrezi fără excluderi explicite.

Din folderul proiectului rulează:

```bash
git add .
git commit -m "Configurare deploy automat cPanel"
git push origin main
```

Apoi deschide **GitHub → Actions → Deploy to cPanel via FTPS** și verifică execuția. Poți porni și manual workflow-ul din **Run workflow**.

După un deploy reușit, verifică în **cPanel → File Manager → public_html** că există direct `index.html` și `assets/` (nu `public_html/dist/index.html`).

## 7. Probleme frecvente

| Eroare | Ce verifici |
| --- | --- |
| `ETIMEDOUT` pe portul 21 către IP-uri Cloudflare | `FTP_SERVER` trebuie să indice serverul FTP real; verifică IP-ul din Server Information sau DNS-only |
| `ETIMEDOUT` către IP-ul real | Portul 21, firewall-ul hostingului, accesul extern FTP și eventualele blocări ale IP-urilor GitHub Actions |
| Eroare certificat FTPS | Folosește hostname-ul direct al serverului acoperit de certificat, configurat DNS-only; nu dezactiva validarea |
| `530 Login incorrect` | Utilizatorul și parola FTP din GitHub Secrets |
| Fișiere în director greșit | Directorul inițial al utilizatorului FTP și `server-dir` |
| `npm ci` eșuează | Existența și sincronizarea fișierului `package-lock.json` |

## Alte tipuri de proiecte

- **HTML/CSS/JavaScript simplu:** elimină pașii Node.js / `npm ci` / build și publică un folder dedicat, de exemplu `local-dir: ./site/`.
- **PHP:** publică numai fișierele necesare site-ului dintr-un folder dedicat; nu publica `.env`, parole sau backupuri.
- **React + PHP:** pregătește un director de deploy care conține atât rezultatul build-ului React, cât și endpoint-urile PHP.
- **Next.js cu server Node.js:** nu funcționează doar prin copiere FTP; este nevoie de hosting cu runtime compatibil. Exportul static poate fi publicat separat.

## Limitări și securitate

Deploy-ul FTP **nu este atomic**: o conexiune întreruptă poate lăsa fișiere actualizate doar parțial. Folosește un utilizator FTP dedicat, limitat la site. Nu salva parole în Git. Pentru fiecare site folosește propriile GitHub Secrets.

## Licență

MIT
