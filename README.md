# Sprawozdanie - Zadanie 1 (Część Obowiązkowa)

**Autor:** Roman Rybak
**Temat:** Konteneryzacja aplikacji Node.js z maksymalną optymalizacją rozmiaru obrazu.

---

## 1. Kod oprogramowania (Serwer Node.js)

Aplikacja została napisana w środowisku Node.js z wykorzystaniem wyłącznie wbudowanych modułów (`http`, `https`), aby zminimalizować rozmiar końcowy (brak katalogu `node_modules`). Serwer po uruchomieniu loguje wymagane dane, a następnie serwuje interfejs webowy do sprawdzania pogody.

```javascript
const http = require('http');
const https = require('https');

// --- KONFIGURACJA ---
const PORT = 8080;
const AUTHOR = "Roman Rybak";

// 1a. Logowanie informacji przy starcie aplikacji (Wymóg zadania)
const startDate = new Date().toLocaleString('pl-PL', { 
    timeZone: 'Europe/Warsaw', 
    dateStyle: 'long', 
    timeStyle: 'medium' 
});

console.log("======================================");
console.log(`Data uruchomienia: ${startDate}`);
console.log(`Autor: ${AUTHOR}`);
console.log(`Aplikacja nasłuchuje na porcie TCP: ${PORT}`);
console.log("======================================");

// Funkcje pomocnicze do mapowania danych z API (poprawa UI)
function getWeatherDescription(code) {
    if (code === 0) return "Bezchmurnie";
    if (code > 0 && code <= 3) return "Zmienne zachmurzenie";
    if (code === 45 || code === 48) return "Mgła";
    if (code >= 51 && code <= 57) return "Mżawka";
    if (code >= 61 && code <= 67) return "Deszcz";
    if (code >= 71 && code <= 77) return "Śnieg";
    if (code >= 80 && code <= 82) return "Ulewa";
    if (code >= 95) return "Burza";
    return "Nieznana";
}

function getWindDirectionLabel(degree) {
    if (degree >= 337.5 || degree < 22.5) return "Północny";
    if (degree >= 22.5 && degree < 67.5) return "Północno-wschodni";
    if (degree >= 67.5 && degree < 112.5) return "Wschodni";
    if (degree >= 112.5 && degree < 157.5) return "Południowo-wschodni";
    if (degree >= 157.5 && degree < 202.5) return "Południowy";
    if (degree >= 202.5 && degree < 247.5) return "Południowo-zachodni";
    if (degree >= 247.5 && degree < 292.5) return "Zachodni";
    return "Północno-zachodni";
}

// Globalne style CSS
const commonCSS = `
<style>
    body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background: linear-gradient(135deg, #74ebd5 0%, #ACB6E5 100%); min-height: 100vh; margin: 0; display: flex; align-items: center; justify-content: center; color: #333; }
    .container { background: rgba(255, 255, 255, 0.95); padding: 30px; border-radius: 15px; box-shadow: 0 10px 30px rgba(0,0,0,0.15); text-align: center; max-width: 400px; width: 90%; }
    select, button { width: 100%; padding: 12px; margin-top: 15px; border-radius: 8px; border: 1px solid #ddd; font-size: 16px; box-sizing: border-box; }
    button { background: #007bff; color: white; border: none; cursor: pointer; font-weight: bold; transition: 0.3s; }
    button:hover { background: #0056b3; }
    h2, h3 { margin-top: 0; color: #2c3e50; }
    .data-row { display: flex; justify-content: space-between; padding: 10px 0; border-bottom: 1px solid #eee; text-align: left; }
    .data-row span { color: #555; }
    .data-row:last-child { border-bottom: none; }
    a { display: inline-block; margin-top: 20px; text-decoration: none; color: #007bff; font-weight: bold; }
    a:hover { text-decoration: underline; }
</style>
`;

// Główny serwer HTTP
const server = http.createServer((req, res) => {
    // 1b. Ekran główny - wybór kraju i miasta
    if (req.url === '/') {
        res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
        res.end(`
            <!DOCTYPE html>
            <html lang="pl">
            <head>
                <meta charset="UTF-8">
                <meta name="viewport" content="width=device-width, initial-scale=1.0">
                <title>Pogoda - Zadanie 1</title>
                ${commonCSS}
            </head>
            <body>
                <div class="container">
                    <h2>Sprawdź pogodę</h2>
                    <form action="/weather" method="GET">
                        <select name="city">
                            <option value="52.23,21.01,Warszawa (Polska)">Polska - Warszawa</option>
                            <option value="51.25,22.57,Lublin (Polska)">Polska - Lublin</option>
                            <option value="51.51,-0.13,Londyn (Wielka Brytania)">Wielka Brytania - Londyn</option>
                            <option value="48.85,2.35,Paryż (Francja)">Francja - Paryż</option>
                            <option value="41.89,12.51,Rzym (Włochy)">Włochy - Rzym</option>
                        </select>
                        <button type="submit">Pokaż wynik</button>
                    </form>
                </div>
            </body>
            </html>
        `);
    } 
    // 1b. Pobieranie danych i wyświetlanie wyników
    else if (req.url.startsWith('/weather?city=')) {
        const cityParam = decodeURIComponent(req.url.split('=')[1]);
        const parts = cityParam.split(',');
        
        if (parts.length !== 3) {
            res.writeHead(400, { 'Content-Type': 'text/html; charset=utf-8' });
            return res.end(`<div class="container"><h3>Błąd parametrów</h3><a href="/">Powrót</a></div>`);
        }

        const lat = parts[0];
        const lon = parts[1];
        const cityName = parts[2];

        const apiUrl = `[https://api.open-meteo.com/v1/forecast?latitude=$](https://api.open-meteo.com/v1/forecast?latitude=$){lat}&longitude=${lon}&current_weather=true&timezone=Europe/Warsaw`;

        https.get(apiUrl, (apiRes) => {
            let data = '';
            apiRes.on('data', (chunk) => data += chunk);
            apiRes.on('end', () => {
                try {
                    const json = JSON.parse(data);
                    if (json.error) throw new Error("Błąd API Open-Meteo");

                    const weather = json.current_weather;
                    const condition = getWeatherDescription(weather.weathercode);
                    const windDirection = getWindDirectionLabel(weather.winddirection);
                    
                    const dateObj = new Date(weather.time);
                    const formattedTime = dateObj.toLocaleString('pl-PL', { dateStyle: 'long', timeStyle: 'short' });

                    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
                    res.end(`
                        <!DOCTYPE html>
                        <html lang="pl">
                        <head>
                            <meta charset="UTF-8">
                            <meta name="viewport" content="width=device-width, initial-scale=1.0">
                            <title>Wynik dla ${cityName}</title>
                            ${commonCSS}
                        </head>
                        <body>
                            <div class="container">
                                <h3>Pogoda: ${cityName}</h3>
                                
                                <div class="data-row">
                                    <span>Aktualizacja:</span>
                                    <b>${formattedTime}</b>
                                </div>
                                <div class="data-row">
                                    <span>Warunki:</span>
                                    <b>${condition}</b>
                                </div>
                                <div class="data-row">
                                    <span>Temperatura:</span>
                                    <b>${weather.temperature} °C</b>
                                </div>
                                <div class="data-row">
                                    <span>Wiatr:</span>
                                    <b>${weather.windspeed} km/h</b>
                                </div>
                                <div class="data-row">
                                    <span>Kierunek wiatru:</span>
                                    <b>${windDirection}</b>
                                </div>
                                
                                <a href="/">← Wybierz inne miasto</a>
                            </div>
                        </body>
                        </html>
                    `);
                } catch (e) {
                    res.writeHead(500, { 'Content-Type': 'text/html; charset=utf-8' });
                    res.end(`<div class="container"><h3>Błąd: ${e.message}</h3><a href="/">Powrót</a></div>`);
                }
            });
        }).on('error', (err) => {
            res.writeHead(500, { 'Content-Type': 'text/html; charset=utf-8' });
            res.end(`<div class="container"><h3>Błąd API: ${err.message}</h3><a href="/">Powrót</a></div>`);
        });
    } else {
        res.writeHead(404, { 'Content-Type': 'text/html; charset=utf-8' });
        res.end(`<!DOCTYPE html><html><head>${commonCSS}</head><body><div class="container"><h3>404 - Nie znaleziono</h3><a href="/">Powrót</a></div></body></html>`);
    }
});

server.listen(PORT);
```
