"use strict";
const { request, Agent } = require("undici");

const TOKEN = "";
const GUILD = "";
const PASSWORDS = ["", "", "", "", "", "", ""]; // 7 adet farklı hesaptan açabilirsiniz kodu (çoklu vanity)

const agent = new Agent({
  pipelining: 10,
  connections: 50,
  keepAliveTimeout: 60000,
  keepAliveMaxTimeout: 600000
});

const IPS = ["162.159.136.232", "162.159.137.232", "162.159.128.233", "162.159.135.232", "162.159.138.232"];
const mfaTokens = new Array(7).fill("");
let vanity = "";
let idx = 0;

const req = async (path, method, body, mfa = "") => {
  try {
    const ip = IPS[idx++ % IPS.length];
    const { statusCode, body: res } = await request(`https://${ip}${path}`, {
      method,
      headers: {
        "Host": "discord.com",
        "Authorization": TOKEN,
        "Content-Type": "application/json",
        "User-Agent": "Mozilla/5.0",
        ...(mfa && { "X-Discord-MFA-Authorization": mfa })
      },
      body: body ? JSON.stringify(body) : undefined,
      dispatcher: agent
    });
    return { status: statusCode, data: await res.json() };
  } catch { return null; }
};

const fetchMfa = async (i) => {
  if (!PASSWORDS[i]) return;
  const ticket = await req("/api/v10/guilds/" + GUILD + "/vanity-url", "PATCH", { code: "" });
  if (!ticket?.data?.mfa?.ticket) return;
  const result = await req("/api/v10/mfa/finish", "POST", {
    ticket: ticket.data.mfa.ticket,
    mfa_type: "password",
    data: PASSWORDS[i]
  });
  if (result?.data?.token) {
    mfaTokens[i] = result.data.token;
    console.log(`[MFA ${i}] ok`);
  }
};

const snipe = async (code) => {
  vanity = code;
  const total = mfaTokens.filter(t => t).length;
  console.log(`[SNIPE] ${code} → ${total} requests`);
  
  const promises = mfaTokens.map((mfa, i) => {
    if (!mfa) return null;
    return req("/api/v10/guilds/" + GUILD + "/vanity-url", "PATCH", { code }, mfa)
      .then(r => {
        if (r?.data?.code === code) {
          console.log(JSON.stringify({ code: r.data.code, uses: r.data.uses || 0 }));
        } else if (r?.data?.message) {
          console.log(JSON.stringify({ message: r.data.message, code: r.data.code }));
        }
      });
  }).filter(Boolean);
  
  await Promise.all(promises);
};

const checkGuilds = async () => {
  const { data } = await req("/api/v10/users/@me/guilds", "GET") || {};
  if (!data) return;
  
  for (const guild of data) {
    if (!guild.vanity_url_code) continue;
    
    const { data: full } = await req(`/api/v10/guilds/${guild.id}`, "GET") || {};
    if (!full?.vanity_url_code) continue;
    
    if (guild.id === GUILD && full.vanity_url_code !== vanity && vanity) {
      await snipe(vanity);
    }
    vanity = full.vanity_url_code;
  }
};

(async () => {
  console.log("[EXA] mfa aldım");
  
  await Promise.all(PASSWORDS.map((_, i) => fetchMfa(i)));
  
  setInterval(() => PASSWORDS.forEach((_, i) => fetchMfa(i)), 300000);
  
  setInterval(checkGuilds, 2000);
})();
