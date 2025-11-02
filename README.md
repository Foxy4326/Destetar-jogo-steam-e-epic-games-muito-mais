// 🎮 GameFreeFinder — Mostra jogos pagos que estão de graça em tempo real
// Autor: Vitor / GPT-5

import express from "express";
import fetch from "node-fetch";
import path from "path";
import { fileURLToPath } from "url";

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
const app = express();
const PORT = process.env.PORT || 3000;

// ===== Middleware para arquivos estáticos =====
app.use(express.static(path.join(__dirname, "public")));

// ===== Funções utilitárias =====
async function buscarJogosGratis() {
  const resultados = [];

  try {
    // 🟦 Steam (via SteamDB API)
    const steamResp = await fetch("https://steamdb.info/api/GetFreeGames/v1/");
    const steamData = await steamResp.json();
    if (steamData && steamData.data) {
      for (const game of steamData.data) {
        resultados.push({
          loja: "Steam",
          titulo: game.title,
          url: `https://store.steampowered.com/app/${game.appid}`,
          fim: game.end_time ? new Date(game.end_time * 1000).toLocaleString() : "Desconhecido",
        });
      }
    }

    // 🟪 Epic Games
    const epicResp = await fetch("https://store-site-backend-static.ak.epicgames.com/freeGamesPromotions?country=BR&locale=pt-BR");
    const epicData = await epicResp.json();
    epicData.data.Catalog.searchStore.elements.forEach((game) => {
      const oferta = game.price.totalPrice.discountPrice === 0;
      if (oferta) {
        resultados.push({
          loja: "Epic Games",
          titulo: game.title,
          url: `https://store.epicgames.com/p/${game.productSlug}`,
          fim: game.promotions?.promotionalOffers?.[0]?.promotionalOffers?.[0]?.endDate
            ? new Date(game.promotions.promotionalOffers[0].promotionalOffers[0].endDate).toLocaleString()
            : "Desconhecido",
        });
      }
    });

    // 🟨 GOG
    const gogResp = await fetch("https://www.gog.com/games/ajax/filtered?mediaType=game&price=free");
    const gogData = await gogResp.json();
    gogData.products.forEach((game) => {
      resultados.push({
        loja: "GOG",
        titulo: game.title,
        url: `https://www.gog.com/game/${game.slug}`,
        fim: "Grátis permanente",
      });
    });

    // 🟥 Prime Gaming (Amazon)
    const primeResp = await fetch("https://gaming.amazon.com/home");
    const primeText = await primeResp.text();
    const regex = /"title":"(.*?)","link":"(.*?)"/g;
    let match;
    while ((match = regex.exec(primeText)) !== null) {
      resultados.push({
        loja: "Prime Gaming",
        titulo: match[1],
        url: `https://gaming.amazon.com${match[2]}`,
        fim: "Disponível por tempo limitado",
      });
    }
  } catch (e) {
    console.error("Erro ao buscar jogos:", e);
  }

  return resultados;
}

// ===== API: retornar lista de jogos =====
app.get("/api/jogos-gratis", async (req, res) => {
  const dados = await buscarJogosGratis();
  res.json(dados);
});

// ===== Página principal =====
app.get("/", (req, res) => {
  res.send(`
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>🎮 GameFreeFinder — Jogos Pagos Grátis</title>
<script src="https://cdn.tailwindcss.com"></script>
<style>
  body { background: linear-gradient(135deg, #0f172a, #1e293b); color: #fff; font-family: 'Inter', sans-serif; }
  .card { background: rgba(255,255,255,0.05); border-radius: 1rem; padding: 1rem; transition: 0.3s; }
  .card:hover { background: rgba(255,255,255,0.1); transform: scale(1.03); }
</style>
</head>
<body class="min-h-screen flex flex-col items-center p-4">
  <h1 class="text-3xl font-bold mb-6 text-center">🎮 Jogos Pagos Grátis Agora</h1>
  <p class="text-gray-300 mb-4 text-center">Atualizado em tempo real (Steam, Epic, GOG, Prime Gaming)</p>

  <div id="jogos" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 w-full max-w-6xl"></div>

  <script>
    async function carregarJogos() {
      const resp = await fetch("/api/jogos-gratis");
      const jogos = await resp.json();
      const container = document.getElementById("jogos");
      container.innerHTML = "";
      if (jogos.length === 0) {
        container.innerHTML = "<p class='text-gray-400 text-center col-span-3'>Nenhuma promoção gratuita no momento 😢</p>";
        return;
      }
      jogos.forEach(j => {
        const el = document.createElement("div");
        el.className = "card";
        el.innerHTML = \`
          <h2 class="text-xl font-bold mb-2">\${j.titulo}</h2>
          <p class="text-sm text-gray-300 mb-2">🛒 Loja: <span class="font-semibold">\${j.loja}</span></p>
          <p class="text-sm text-gray-400 mb-2">⏰ Até: \${j.fim}</p>
          <a href="\${j.url}" target="_blank" class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-full text-sm">🎁 Resgatar Grátis</a>
        \`;
        container.appendChild(el);
      });
    }

    carregarJogos();
    setInterval(carregarJogos, 60000); // Atualiza a cada 60 segundos
  </script>
</body>
</html>
  `);
});

// ===== Iniciar servidor =====
app.listen(PORT, () => console.log("🚀 Servidor rodando em http://localhost:" + PORT));
