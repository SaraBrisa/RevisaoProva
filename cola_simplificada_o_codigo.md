📒 COLA — API + App do zero · prova_2ams_<nome-sobrenome>
Stack: Node + Express + MySQL (API) · Expo + React Native + Expo Router (App) · JWT

Como ler

Marca	Significa
🧠	por que o código é assim (explicação do detalhe)
♻	repetição: o arquivo é igual a outro, e a marca diz só o que muda
📦	arquivo que você leva pronto de casa (Parte B)
✍	arquivo que você digita na hora
⚠	erro clássico
0. Visão geral (guarde isto na cabeça)
0.1 Fluxo
Registrar ─▶ POST /auth/register ─▶ grava usuário com senha em HASH (bcrypt)
Login     ─▶ POST /auth/login    ─▶ confere hash ─▶ devolve TOKEN (JWT, 1h) + user
App guarda {token,user} no AsyncStorage ─▶ TODA chamada seguinte leva "Authorization: Bearer <token>"
Cadastros ─▶ GET/POST/PUT/DELETE /<tema> ─▶ API só responde se o token for válido
Sair      ─▶ apaga a sessão ─▶ o "porteiro" do app manda para o login sozinho
0.2 Contrato único de resposta da API (tudo o que a API devolve tem este formato)
{ "ok": true, "message": "Success", "data": ... }
O app só olha o status HTTP (200–299 = deu certo) e lê message e data. Assim uma tela nunca precisa conhecer detalhes da API.

0.3 O que foi padronizado (e por que a prova fica mais rápida)
Repetição típica	Como foi resolvida	Custo de um tema novo
1 rota + model + controller por tabela na API	uma fábrica crud(nome) gera as 5 rotas REST de qualquer tabela	1 bloco em recursos.js
1 arquivo de API por tabela no app	criarApi("endpoint") devolve listar/criar/editar/excluir	1 linha em temas.ts
6 telas de cadastro quase idênticas	um componente TelaCadastro recebe um objeto "tema"	arquivo de 3 linhas
Itens do menu Drawer escritos à mão	gerados com .map sobre temas	nada (automático)
Login e Register com inputs repetidos	componente CampoTexto (rótulo + input)	—
Redirecionar após login/logout em cada tela	um porteiro no _layout raiz (Stack.Protected)	—
0.4 Legenda da árvore (suas siglas)
Sigla	Descrição
nome/	pasta
(nome)/	pasta de agrupamento de rota (não aparece na URL)
nome.tsx	arquivo de tela/rota
(NomeComponente)	componente visual exportado pelo arquivo
0.5 Árvore final
prova_2ams_<nome-sobrenome>/
├── .gitignore
├── api/
│   ├── .env
│   └── src/
│       ├── server.js        db.js        recursos.js     helpers.js
│       ├── crud.js          auth.js      middleware.js   routes.js
│       └── tests/usersTest.http
└── <app-nome-do-app>/
    └── src/
        ├── api/            apiConfig.ts 📦   apiAuth.ts ✍   apiUsers.ts ✍
        ├── components/     ButtonFatec.tsx (ButtonFatec) 📦   LogoApp.tsx (LogoApp) 📦
        │                   CampoTexto.tsx (CampoTexto) 📦     TelaCadastro.tsx (TelaCadastro) 📦
        ├── config/         temas.ts ✍
        ├── styles/         estilos.ts 📦
        ├── utils/          AuthContext.tsx (AuthProvider) 📦
        └── app/
            ├── _layout.tsx (RootLayout) ✍
            ├── login.tsx (Login) ✍        register.tsx (Register) ✍
            └── (auth)/
                ├── _layout.tsx (Abas) ✍
                ├── index.tsx (Home) ✍      perfil.tsx (PerfilUser) ✍      sair.tsx (SairApp) ✍
                └── (cadastros)/
                    ├── _layout.tsx (Menu Drawer) ✍
                    └── clientes · fornecedores · estados · cidades · roles · users  (.tsx) ✍
🧠 Por que components/, utils/, styles/, config/ ficam FORA de src/app/? No Expo Router todo arquivo dentro de app/ vira uma rota (e gera avisos de "sem default export"). Só telas e _layout moram em app/. Isso é a única diferença em relação ao desenho das suas anotações, em que utils/ e components/ aparecem dentro de (auth)/.

1. Checklist e criação do projeto
Item	Valor
Pasta	prova_2ams_<nome-sobrenome>/ com api/ e <app-nome-do-app>/
Banco	<nome-do-banco-novo> (a API cria sozinha)
Porta da API	a da prova (aqui 3500), igual em: api/.env, usersTest.http e apiConfig.ts
DB_RESET	true na 1ª execução → false logo depois
Pré-requisito	MySQL ligado (XAMPP / serviço / Workbench)
mkdir prova_2ams_nome-sobrenome && cd prova_2ams_nome-sobrenome
printf "node_modules/\n.env\n" > .gitignore
mkdir api
npx create-expo-app@latest <app-nome-do-app>
cd <app-nome-do-app> && npm run reset-project      # responda o prompt conforme sua anotação (R)
🧠 O .gitignore impede subir node_modules e o .env (senha do banco e chave do JWT). Confirme que as telas ficam em src/app (mova se o reset criar app/ na raiz).

PARTE A — API (api/)
A.1 Projeto Node ✍
cd api && npm init -y
npm install express cors dotenv bcryptjs jsonwebtoken mysql2
mkdir -p src/tests
No package.json acrescente/ajuste:

"type": "module",
"scripts": { "dev": "node --watch src/server.js" }
🧠

"type": "module" liga import/export e await no topo do arquivo (usado em db.js).
--watch reinicia a API ao salvar um arquivo (dispensa nodemon). ⚠ ele NÃO reinicia ao mudar o .env: pare (Ctrl+C) e rode de novo.
cors só é necessário se você testar o app no navegador (npm run web); no celular não faz diferença.
bcryptjs = hash de senha · jsonwebtoken = token · mysql2 = driver do banco (versão com Promise).
api/.env:

API_PORT=3500
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=
DB_DATABASE=<nome-do-banco-novo>
DB_RESET=true
JWT_SECRET=troque-por-uma-frase-longa
🧠

Configuração fica fora do código: mudar porta/banco não exige editar arquivo .js.
⚠ DB_RESET=true faz DROP TABLE + CREATE TABLE. Como o --watch reinicia a cada salvamento, deixar true apaga seus dados toda vez que você salvar qualquer arquivo. Rodou a 1ª vez? Mude para false.
DB_PASSWORD= vazio é o padrão do root local (XAMPP). Se o seu MySQL tem senha, preencha.
A.2 src/recursos.js ✍ — a única fonte de verdade das tabelas
// Cada chave = uma TABELA (e também o endereço da rota: /clientes, /cidades ...).
// "colunas" = nome da coluna → definição SQL. O id é criado automaticamente.
export const recursos = {
  users: {
    colunas: {
      name: "VARCHAR(100) NOT NULL",
      login: "VARCHAR(100) NOT NULL UNIQUE", // UNIQUE = o banco recusa login repetido
      password: "VARCHAR(255) NOT NULL",     // 255 porque guarda o HASH (não a senha)
    },
  },
  roles: { colunas: { role: "VARCHAR(60) NOT NULL" } },
  clientes: {
    colunas: {
      nome: "VARCHAR(100) NOT NULL",
      email: "VARCHAR(150) NOT NULL",
      telefone: "VARCHAR(20) NULL",
    },
  },
  fornecedores: {
    colunas: {
      nome: "VARCHAR(100) NOT NULL",
      cnpj: "VARCHAR(20) NULL",
      telefone: "VARCHAR(20) NULL",
      email: "VARCHAR(150) NULL",
    },
  },
  estados: {
    colunas: { estado: "VARCHAR(80) NOT NULL", uf: "CHAR(2) NOT NULL", regiao: "VARCHAR(30) NULL" },
  },
  cidades: {
    colunas: { cidade: "VARCHAR(100) NOT NULL", uf: "CHAR(2) NOT NULL", populacao: "INT NULL" },
  },
};
🧠 Este objeto é lido em 3 lugares: db.js (monta o CREATE TABLE), crud.js (lista de colunas permitidas, ver A.5) e routes.js (cria uma rota por chave). Tema novo na API = 1 bloco aqui e mais nada. Os nomes das colunas devem ser idênticos ao campo chave que você usará no temas.ts do app.

A.3 src/db.js ✍ — conexão + criação do banco e das tabelas
import mysql from "mysql2/promise";
import { recursos } from "./recursos.js";

const { DB_HOST, DB_USER, DB_PASSWORD, DB_DATABASE, DB_RESET } = process.env;

// 1) Conexão SEM "database": o banco pode ainda não existir.
const admin = await mysql.createConnection({ host: DB_HOST, user: DB_USER, password: DB_PASSWORD });
await admin.query(`CREATE DATABASE IF NOT EXISTS \`${DB_DATABASE}\` CHARACTER SET utf8mb4`);
await admin.end();

// 2) Pool = várias conexões reaproveitadas (mais rápido que abrir uma por requisição).
export const pool = mysql.createPool({
  host: DB_HOST, user: DB_USER, password: DB_PASSWORD, database: DB_DATABASE,
});

// 3) Para cada recurso: (opcional) apaga e recria a tabela.
for (const [tabela, { colunas }] of Object.entries(recursos)) {
  if (DB_RESET === "true") await pool.query(`DROP TABLE IF EXISTS \`${tabela}\``);
  const defs = Object.entries(colunas).map(([nome, def]) => `\`${nome}\` ${def}`).join(", ");
  await pool.query(
    `CREATE TABLE IF NOT EXISTS \`${tabela}\` (id INT AUTO_INCREMENT PRIMARY KEY, ${defs}) DEFAULT CHARSET=utf8mb4`
  );
}
console.log("[db] pronto:", Object.keys(recursos).join(", "));
🧠

mysql2/promise → usamos await em vez de callbacks.
query × execute: query para comandos de estrutura (DDL: CREATE/DROP); execute (usado no crud.js) para comandos com parâmetros ?, que são enviados separados do SQL e impedem SQL Injection.
As crases ` em volta de nomes protegem palavras reservadas.
IF NOT EXISTS deixa o código idempotente: rodar de novo (com DB_RESET=false) não quebra nem apaga nada.
DB_RESET === "true" é comparação de texto: tudo que vem do .env é string.
O await solto no topo só funciona por causa do "type": "module".
A.4 src/helpers.js ✍ — resposta padrão e captura de erro
// Toda resposta da API sai por aqui → contrato único { ok, message, data }.
export const enviar = (res, status, message, data) =>
  res.status(status).json({ ok: status < 400, message, data });

// Envolve um handler async: qualquer erro vira resposta HTTP em vez de derrubar o servidor.
export const tratar = (fn) => async (req, res) => {
  try {
    await fn(req, res);
  } catch (e) {
    if (e.code === "ER_DUP_ENTRY") return enviar(res, 409, "Registro duplicado: valor já cadastrado");
    enviar(res, 400, e.message);
  }
};
🧠

ok: status < 400 é calculado sozinho; ninguém esquece de preencher.
ER_DUP_ENTRY é o erro do MySQL quando viola o UNIQUE (ex.: login repetido) → responde 409 Conflict.
Campos NOT NULL faltando também caem no catch e viram 400 com a mensagem do banco (útil para depurar na prova).
Sem o tratar, uma exceção dentro de uma rota async deixaria a requisição pendurada.
A.5 src/crud.js ✍ — a fábrica: 5 rotas REST para qualquer tabela
import { Router } from "express";
import { pool } from "./db.js";
import { recursos } from "./recursos.js";
import { enviar, tratar } from "./helpers.js";

// Camadas (MVC) juntas em 1 arquivo: consultas = Model · handlers = Controller · Router = View/rotas
export function crud(nome, { omitir = [], preparar = async (b) => b } = {}) {
  const tabela = `\`${nome}\``;
  const campos = Object.keys(recursos[nome].colunas);        // LISTA BRANCA de colunas

  // remove do resultado as colunas sensíveis (ex.: password)
  const limpar = (linha) => Object.fromEntries(Object.entries(linha).filter(([k]) => !omitir.includes(k)));

  // transforma o body em pares [coluna, valor], aceitando SÓ colunas conhecidas
  const escolher = async (body) => {
    const dados = await preparar({ ...body });
    return campos.filter((c) => dados[c] !== undefined).map((c) => [c, dados[c]]);
  };

  const router = Router();

  router.get(`/${nome}`, tratar(async (req, res) => {
    const [linhas] = await pool.execute(`SELECT * FROM ${tabela} ORDER BY id`);
    enviar(res, 200, "Success", linhas.map(limpar));
  }));

  router.get(`/${nome}/:id`, tratar(async (req, res) => {
    const [linhas] = await pool.execute(`SELECT * FROM ${tabela} WHERE id = ?`, [req.params.id]);
    if (!linhas.length) return enviar(res, 404, "Não encontrado");
    enviar(res, 200, "Success", limpar(linhas[0]));
  }));

  router.post(`/${nome}`, tratar(async (req, res) => {
    const pares = await escolher(req.body);
    if (!pares.length) return enviar(res, 400, "Nenhum campo válido no body");
    const cols = pares.map(([c]) => `\`${c}\``).join(", ");
    const marcas = pares.map(() => "?").join(", ");
    const [r] = await pool.execute(`INSERT INTO ${tabela} (${cols}) VALUES (${marcas})`, pares.map(([, v]) => v));
    enviar(res, 201, "Success", { id: r.insertId });
  }));

  router.put(`/${nome}/:id`, tratar(async (req, res) => {
    const pares = await escolher(req.body);
    if (!pares.length) return enviar(res, 400, "Nenhum campo válido no body");
    const sets = pares.map(([c]) => `\`${c}\` = ?`).join(", ");
    const [r] = await pool.execute(`UPDATE ${tabela} SET ${sets} WHERE id = ?`, [...pares.map(([, v]) => v), req.params.id]);
    if (!r.affectedRows) return enviar(res, 404, "Não encontrado");
    enviar(res, 200, "Success");
  }));

  router.delete(`/${nome}/:id`, tratar(async (req, res) => {
    const [r] = await pool.execute(`DELETE FROM ${tabela} WHERE id = ?`, [req.params.id]);
    if (!r.affectedRows) return enviar(res, 404, "Não encontrado");
    enviar(res, 200, "Success");
  }));

  return router;
}
🧠

Lista branca (campos): o body vem do cliente. Só entram colunas que existem em recursos.js. Sem isso, alguém mandaria {"id": 1} (sobrescrever id) ou um nome de coluna malicioso (os ? protegem só os valores, nunca os nomes).
omitir e preparar são os ganchos para casos especiais (só users usa; ver A.8). Valores padrão fazem os outros temas funcionarem sem configurar nada.
escolher ignora chaves undefined → um PUT parcial (só {nome}) atualiza apenas aquela coluna.
Status: 201 no POST (criado), 404 quando o id não existe (affectedRows = quantas linhas foram atingidas), 200 no resto.
execute não aceita undefined como parâmetro; por isso o filtro !== undefined.
A.6 src/auth.js ✍ — registro e login
import bcrypt from "bcryptjs";
import jwt from "jsonwebtoken";
import { Router } from "express";
import { pool } from "./db.js";
import { enviar, tratar } from "./helpers.js";

const router = Router();

router.post("/register", tratar(async (req, res) => {
  const { name, login, password } = req.body ?? {};
  if (!name || !login || !password) return enviar(res, 400, "Informe name, login e password");
  const hash = await bcrypt.hash(password, 10);          // 10 = custo do algoritmo
  await pool.execute("INSERT INTO users (name, login, password) VALUES (?, ?, ?)", [name, login, hash]);
  enviar(res, 201, "Usuário criado");
}));

router.post("/login", tratar(async (req, res) => {
  const { login, password } = req.body ?? {};
  if (!login || !password) return enviar(res, 400, "Informe login e password");
  const [[u]] = await pool.execute("SELECT id, name, password FROM users WHERE login = ?", [login]);
  if (!u || !(await bcrypt.compare(password, u.password)))
    return enviar(res, 401, "Login ou senha inválidos");   // mesma mensagem nos 2 casos
  const token = jwt.sign({ id: u.id }, process.env.JWT_SECRET, { expiresIn: "1h" });
  enviar(res, 200, "Success", { token, user: { id: u.id, name: u.name } });
}));

export default router;
🧠

Nunca se guarda a senha: bcrypt.hash gera um hash irreversível (com "sal" embutido); bcrypt.compare confere a senha digitada contra o hash.
Mensagem igual para "login não existe" e "senha errada" → quem ataca não descobre quais logins existem.
const [[u]] = execute devolve [linhas, metadados]; o 2º par de colchetes pega a 1ª linha (fica undefined se não achou).
O JWT carrega só o id (nunca senha) e expira em 1 h; é assinado com o JWT_SECRET do .env.
Login repetido no register cai no UNIQUE → tratar responde 409.
req.body ?? {} evita erro quando o cliente não manda JSON.
A.7 src/middleware.js ✍ — o porteiro
import jwt from "jsonwebtoken";
import { enviar } from "./helpers.js";

export function exigirToken(req, res, next) {
  const token = req.headers.authorization?.split(" ")[1];      // "Bearer <token>" → pega o 2º pedaço
  if (!token) return enviar(res, 401, "Token não informado");
  try {
    req.userId = jwt.verify(token, process.env.JWT_SECRET).id;  // lança erro se inválido/expirado
    next();                                                     // libera a rota seguinte
  } catch {
    enviar(res, 401, "Token inválido ou expirado");
  }
}
🧠 Middleware = função que roda antes da rota. Sem next() a requisição para aqui. O app usa o 401 para saber que a sessão expirou (ver http na Parte B).

A.8 src/routes.js ✍ — junta tudo
import bcrypt from "bcryptjs";
import { Router } from "express";
import { crud } from "./crud.js";
import { recursos } from "./recursos.js";
import authRouter from "./auth.js";
import { exigirToken } from "./middleware.js";

const router = Router();

router.use("/auth", authRouter);      // PÚBLICAS: /auth/login e /auth/register
router.use(exigirToken);              // ↓ tudo abaixo exige token (comente esta linha se a prova não pedir JWT)

// Único caso especial: users (esconde o hash e transforma a senha nova em hash)
router.use(crud("users", {
  omitir: ["password"],
  preparar: async (body) => {
    if (body.password) body.password = await bcrypt.hash(body.password, 10);
    else delete body.password;         // senha vazia no PUT = manter a antiga
    return body;
  },
}));

// ♻ TODOS os outros temas: 1 linha para o loop inteiro
Object.keys(recursos)
  .filter((nome) => nome !== "users")
  .forEach((nome) => router.use(crud(nome)));

export default router;
🧠

A ordem manda: /auth vem antes de exigirToken, senão ninguém conseguiria logar.
♻ O forEach repete a mesma fábrica para roles, clientes, fornecedores, estados, cidades (e para qualquer chave nova de recursos.js).
users usa os ganchos: omitir (o hash nunca sai da API) e preparar (senha nova sempre entra como hash).
A.9 src/server.js ✍
import "dotenv/config";                 // 1º import: carrega o .env antes de todo o resto
import express from "express";
import cors from "cors";
import os from "os";
import "./db.js";                       // só de importar, cria banco e tabelas
import routes from "./routes.js";
import { enviar } from "./helpers.js";

const porta = process.env.API_PORT || 3500;
const app = express();

app.use(cors());
app.use(express.json());                // sem isto, req.body chega undefined
app.get("/test", (req, res) => enviar(res, 200, "servidor rodando", { porta }));
app.use(routes);
app.use((req, res) => enviar(res, 404, "Rota não encontrada"));   // por último

const ip = Object.values(os.networkInterfaces()).flat()
  .find((i) => i?.family === "IPv4" && !i.internal)?.address ?? "localhost";
app.listen(porta, () => console.log(`API em http://${ip}:${porta}`));
🧠

import "dotenv/config" precisa ser o primeiro: db.js lê process.env assim que é importado.
O 404 fica depois das rotas: só pega o que ninguém tratou.
O IP impresso é o da sua máquina na rede (útil para conferir; o app o descobre sozinho, ver apiConfig.ts).
Rode: cd api && npm run dev → deve aparecer [db] pronto: .... Agora mude DB_RESET=false e reinicie.

A.10 src/tests/usersTest.http ✍ (extensão REST Client do VS Code)
@base = http://localhost:3500

### 1) registrar
POST {{base}}/auth/register
Content-Type: application/json

{ "name": "Teste", "login": "teste", "password": "123456" }

### 2) login (guarda a resposta com o nome "login")
# @name login
POST {{base}}/auth/login
Content-Type: application/json

{ "login": "teste", "password": "123456" }

### 3) extrai o token da resposta
@token = {{login.response.body.data.token}}

### 4) rota protegida
GET {{base}}/clientes
Authorization: Bearer {{token}}

### 5) criar cliente
POST {{base}}/clientes
Authorization: Bearer {{token}}
Content-Type: application/json

{ "nome": "Ana", "email": "ana@x.com", "telefone": "1499999" }
🧠 Testa a API sozinha antes de abrir o app: se falha aqui, o problema é a API. ⚠ Confira a porta na 1ª linha. Sem o header Authorization o passo 4 deve dar 401 (isso prova que o porteiro funciona).

PARTE B — Arquivos do app que você leva PRONTOS 📦
Instale antes (dentro da pasta do app):

npx expo install @react-native-async-storage/async-storage @react-navigation/drawer
🧠 expo install escolhe a versão compatível com o seu SDK. O Drawer não vem no template: sem @react-navigation/drawer, o import ... from "expo-router/drawer" quebra. expo-constants, @expo/vector-icons, gesture-handler e reanimated já vêm no template.

B.1 src/styles/estilos.ts 📦 — cores e estilos das telas (o "CSS")
import { StyleSheet } from "react-native";

export const cores = {
  primaria: "#1D4ED8",
  primariaClara: "#DBEAFE",
  destaque: "#F59E0B",
  perigo: "#DC2626",
  fundo: "#F8FAFC",
  branco: "#FFFFFF",
  texto: "#0F172A",
  suave: "#64748B",
  borda: "#E2E8F0",
};

export const estilos = StyleSheet.create({
  // ---- fundos de tela ----
  tela: { flex: 1, backgroundColor: cores.fundo, padding: 16 },
  telaCentro: { flex: 1, backgroundColor: cores.fundo, padding: 24, alignItems: "center", justifyContent: "center" },
  telaAuth: { flexGrow: 1, backgroundColor: cores.fundo, padding: 24, paddingTop: 60, alignItems: "center" },

  // ---- textos ----
  titulo: { fontSize: 30, fontWeight: "700", color: cores.primaria, marginTop: 20 },
  subtitulo: { color: cores.suave, marginTop: 8, marginBottom: 24, textAlign: "center" },
  link: { color: cores.destaque, fontWeight: "700" },

  // ---- formulário ----
  campo: { width: "100%", marginBottom: 16 },
  rotuloCampo: { fontSize: 12, fontWeight: "600", color: cores.suave, marginBottom: 4 },
  input: {
    width: "100%", padding: 12, borderRadius: 10, fontSize: 16,
    color: cores.texto, backgroundColor: cores.branco, borderWidth: 1, borderColor: cores.borda,
  },

  // ---- cartão de informação / item de lista ----
  card: { backgroundColor: cores.branco, borderRadius: 12, borderWidth: 1, borderColor: cores.borda, padding: 14 },
  cardRotulo: { fontSize: 12, color: cores.suave, marginTop: 6 },
  cardValor: { fontSize: 16, color: cores.texto },

  // ---- barras e botões pequenos ----
  barraTopo: { flexDirection: "row", justifyContent: "space-between", alignItems: "center", marginBottom: 12 },
  acoes: { flexDirection: "row", gap: 8, marginTop: 10 },
  botaoMini: { width: 46, height: 36, marginTop: 0, borderRadius: 10 },
});
🧠

Um arquivo só com as cores: mudar o visual = mudar cores.
StyleSheet.create valida os estilos e permite reutilizar objetos por nome (estilos.card).
Os fundos (tela, telaCentro, telaAuth) existem porque cada tipo de tela alinha o conteúdo de um jeito: lista (tela), conteúdo centralizado (telaCentro) e formulários com scroll (telaAuth usa flexGrow porque vai dentro de um ScrollView).
gap (espaço entre filhos) funciona em flexDirection: "row" nas versões atuais do React Native.
B.2 src/api/apiConfig.ts 📦 — endereço, token e fetch (a única função com fetch do app)
import Constants from "expo-constants";

// ================= 1) ENDEREÇO =================
const PORTA = 3500;        // ⚠ mesma do API_PORT
const IP_MANUAL = "";      // só preencha se a detecção automática falhar. Ex.: "192.168.0.25"

// hostUri = "192.168.0.25:8081" (endereço do servidor do Expo). Pegamos só o IP antes dos ":".
const host = IP_MANUAL || Constants.expoConfig?.hostUri?.split(":")[0] || "localhost";
export const apiUri = `http://${host}:${PORTA}`;
export const AUTH_KEY = "auth-key";        // chave do AsyncStorage (usada pelo AuthContext)

// ================= 2) TOKEN EM MEMÓRIA =================
let tokenAtual: string | null = null;
let aoExpirar: (() => void) | null = null;
export const definirToken = (t: string | null) => { tokenAtual = t; };
export const aoSessaoExpirar = (fn: () => void) => { aoExpirar = fn; };

// ================= 3) FETCH PADRÃO =================
export type Registro = Record<string, any>;
export interface Resposta<T = any> { ok: boolean; status: number; message: string; data: T; body: any }

export async function http<T = any>(
  caminho: string,
  opcoes: { method?: string; body?: unknown } = {},
): Promise<Resposta<T>> {
  try {
    const r = await fetch(`${apiUri}${caminho}`, {
      method: opcoes.method ?? "GET",
      headers: {
        "Content-Type": "application/json",
        ...(tokenAtual ? { Authorization: `Bearer ${tokenAtual}` } : {}),
      },
      body: opcoes.body ? JSON.stringify(opcoes.body) : undefined,
    });
    const corpo = await r.json().catch(() => ({}));            // resposta sem JSON não derruba
    if (r.status === 401 && tokenAtual) aoExpirar?.();         // sessão venceu → avisa o contexto
    return { ok: r.ok, status: r.status, message: corpo.message ?? corpo.Error ?? "", data: corpo.data, body: corpo };
  } catch {
    return { ok: false, status: 0, message: "Sem conexão com a API.", data: undefined as T, body: null };
  }
}

// ================= 4) FÁBRICA DE API (♻ 1 chamada por tema) =================
export const criarApi = (recurso: string) => ({
  listar: () => http<Registro[]>(`/${recurso}`),
  criar: (dados: Registro) => http(`/${recurso}`, { method: "POST", body: dados }),
  editar: (id: number, dados: Registro) => http(`/${recurso}/${id}`, { method: "PUT", body: dados }),
  excluir: (id: number) => http(`/${recurso}/${id}`, { method: "DELETE" }),
});
export type Api = ReturnType<typeof criarApi>;
🧠

IP automático: no Expo Go o celular já sabe o IP do PC (é por ele que carrega o app). Isso evita o erro nº 1 da prova (IP digitado errado). ⚠ Se você usa expo start --tunnel, o host vira um domínio do túnel: preencha IP_MANUAL. Emulador Android também pode precisar de "10.0.2.2". localhost no celular aponta para o próprio celular.
Token em variável de módulo (não em useState): http é uma função comum, fora de componentes React, e precisa ler o token sem hooks. O AuthContext chama definirToken ao abrir o app, logar e sair.
fetch NÃO lança erro em 400/401/500: só lança quando a rede falha. Por isso lemos r.ok. O catch cobre a rede caída e devolve o mesmo formato de resposta, então nenhuma tela precisa de try/catch.
body guarda o JSON completo da resposta (útil se a API do professor tiver outro formato, ver E.2). corpo.message ?? corpo.Error aceita também APIs que usam Error. Content-Type: application/json é obrigatório: sem ele o express.json() não lê o body.
401 + tokenAtual: só considera "sessão expirada" quando havia token. No login com senha errada não havia token, então não desloga ninguém.
criarApi espelha a fábrica crud da API (5 rotas ↔ 4 funções; o "buscar por id" não é usado nas telas, por isso foi omitido).
B.3 src/utils/AuthContext.tsx 📦
import { AuthLogin } from "@/api/apiAuth";
import { AUTH_KEY, aoSessaoExpirar, definirToken } from "@/api/apiConfig";
import AsyncStorage from "@react-native-async-storage/async-storage";
import { SplashScreen } from "expo-router";
import { createContext, PropsWithChildren, useCallback, useContext, useEffect, useState } from "react";
import { Alert } from "react-native";

SplashScreen.preventAutoHideAsync();        // segura a splash até sabermos se há sessão salva

export interface User { id: number; nome: string; email: string }
interface Sessao { token: string; user: User }
interface Ctx {
  user: User | null;
  isLoggedIn: boolean;
  isReady: boolean;
  logIn: (login: string, senha: string) => Promise<string | null>;   // null = deu certo; texto = mensagem de erro
  logOut: () => Promise<void>;
}

export const AuthContext = createContext<Ctx>({} as Ctx);
export const useAuth = () => useContext(AuthContext);

export default function AuthProvider({ children }: PropsWithChildren) {
  const [sessao, setSessao] = useState<Sessao | null>(null);
  const [isReady, setIsReady] = useState(false);

  // Único ponto que altera a sessão: memória + token da API + AsyncStorage
  const aplicar = useCallback(async (nova: Sessao | null) => {
    definirToken(nova?.token ?? null);
    setSessao(nova);
    try {
      if (nova) await AsyncStorage.setItem(AUTH_KEY, JSON.stringify(nova));
      else await AsyncStorage.removeItem(AUTH_KEY);
    } catch { console.log("falha ao gravar a sessão"); }
  }, []);

  // Ao abrir o app: recupera a sessão salva
  useEffect(() => {
    (async () => {
      try {
        const bruto = await AsyncStorage.getItem(AUTH_KEY);
        if (bruto) { const s: Sessao = JSON.parse(bruto); definirToken(s.token); setSessao(s); }
      } catch { /* storage ilegível: começa deslogado */ }
      setIsReady(true);
    })();
  }, []);

  useEffect(() => { if (isReady) SplashScreen.hideAsync(); }, [isReady]);

  // Token venceu (a API respondeu 401): desloga
  useEffect(() => {
    aoSessaoExpirar(() => { Alert.alert("Sessão expirada", "Faça login novamente."); aplicar(null); });
  }, [aplicar]);

  async function logIn(login: string, senha: string) {
    const r = await AuthLogin(login, senha);
    if (!r.ok) return r.message;
    await aplicar({ token: r.token, user: { id: r.user.id, nome: r.user.name, email: login } });
    return null;
  }
  const logOut = () => aplicar(null);

  return (
    <AuthContext.Provider value={{ user: sessao?.user ?? null, isLoggedIn: sessao !== null, isReady, logIn, logOut }}>
      {children}
    </AuthContext.Provider>
  );
}
🧠

Este arquivo NÃO navega. Quem redireciona é o porteiro do _layout.tsx raiz (Stack.Protected, ver C.1): mudou isLoggedIn → o roteador troca de tela sozinho. Por isso as telas não chamam router.replace após logar/sair.
isLoggedIn é derivado (sessao !== null): impossível ficar "logado sem token".
aplicar é o único lugar que grava; logIn e logOut só chamam ele → nunca dá para esquecer de limpar o token da API.
logIn devolve string | null (mensagem de erro ou nada): a tela decide só se mostra o alerta.
isReady existe porque ler o AsyncStorage é assíncrono: sem ele o app acharia que você está deslogado por alguns milissegundos.
useCallback mantém aplicar com a mesma identidade entre renders (evita o useEffect rodar de novo).
export const AuthContext também é exportado porque o enunciado pode pedir useContext(AuthContext); useAuth() é só um atalho.
B.4 src/components/ButtonFatec.tsx 📦
import { cores } from "@/styles/estilos";
import { ReactNode } from "react";
import { StyleProp, StyleSheet, Text, TextStyle, TouchableOpacity, ViewStyle } from "react-native";

interface Props {
  onFunctionButton?: () => void;
  titleButton?: string;
  icon?: ReactNode;
  styleButton?: StyleProp<ViewStyle>;
  styleTitle?: StyleProp<TextStyle>;
  disabled?: boolean;
}

export default function ButtonFatec({ onFunctionButton, titleButton, icon, styleButton, styleTitle, disabled }: Props) {
  return (
    <TouchableOpacity
      onPress={() => onFunctionButton?.()}
      disabled={disabled}
      activeOpacity={0.8}
      style={[s.botao, disabled && { opacity: 0.5 }, styleButton]}
    >
      {icon}
      {titleButton ? <Text style={[s.texto, icon ? { marginLeft: 8 } : null, styleTitle]}>{titleButton}</Text> : null}
    </TouchableOpacity>
  );
}

const s = StyleSheet.create({
  botao: {
    backgroundColor: cores.primaria, width: "80%", height: 46, marginTop: 24,
    borderRadius: 14, flexDirection: "row", alignItems: "center", justifyContent: "center",
  },
  texto: { color: "#fff", fontSize: 18, fontWeight: "600" },
});
🧠

[estilo base, estilo condicional, estilo de fora]: o RN aceita lista de estilos e o último vence → styleButton={{ backgroundColor: "red" }} troca só a cor.
onPress={() => onFunctionButton?.()}: ?.() chama só se existir (prop opcional).
{icon} sem ícone não desenha nada; a margem do texto só existe com ícone (evita texto torto).
⚠ Ao passar função com argumento: onFunctionButton={() => apagar(item)} (não {apagar(item)}, que executa na hora).
B.5 src/components/LogoApp.tsx 📦
import { cores } from "@/styles/estilos";
import { Image, StyleSheet, View } from "react-native";

export default function LogoApp() {
  return (
    <View style={s.caixa}>
      <Image source={require("@/assets/images/icon.png")} style={s.imagem} alt="Logo do aplicativo" />
    </View>
  );
}

const s = StyleSheet.create({
  caixa: { backgroundColor: cores.primariaClara, padding: 10, borderRadius: 25 },
  imagem: { width: 100, height: 100, borderRadius: 18 },
});
🧠 @/assets/... funciona por causa do alias no tsconfig.json ("@/assets/*": ["./assets/*"]) → aponta para a pasta assets/ na raiz do app. Para outra imagem, troque só o caminho do require.

B.6 src/components/CampoTexto.tsx 📦 — rótulo + input (♻ usado em Login, Register e no formulário dos cadastros)
import { cores, estilos } from "@/styles/estilos";
import { Text, TextInput, TextInputProps, View } from "react-native";

export default function CampoTexto({ rotulo, style, ...props }: TextInputProps & { rotulo: string }) {
  return (
    <View style={estilos.campo}>
      <Text style={estilos.rotuloCampo}>{rotulo}</Text>
      <TextInput
        style={[estilos.input, style]}
        placeholderTextColor={cores.suave}
        autoCapitalize="none"
        {...props}
      />
    </View>
  );
}
🧠

TextInputProps & { rotulo } = aceita todas as props de um TextInput (value, onChangeText, secureTextEntry, keyboardType, maxLength...) mais o rotulo.
...props vem depois de autoCapitalize="none", então quem usa pode sobrescrever (autoCapitalize="words" no nome).
B.7 src/components/TelaCadastro.tsx 📦 — uma tela que serve para todos os cadastros
import type { Api, Registro } from "@/api/apiConfig";
import { cores, estilos } from "@/styles/estilos";
import Ionicons from "@expo/vector-icons/Ionicons";
import { useFocusEffect } from "expo-router";
import { ComponentProps, useCallback, useState } from "react";
import { Alert, FlatList, KeyboardTypeOptions, Modal, ScrollView, Text, View } from "react-native";
import ButtonFatec from "./ButtonFatec";
import CampoTexto from "./CampoTexto";

// ---- tipos que o temas.ts usa ----
export interface Campo {
  chave: string;                 // nome EXATO da coluna em recursos.js
  rotulo: string;                // texto mostrado ao usuário
  teclado?: KeyboardTypeOptions; // "email-address", "phone-pad"...
  numero?: boolean;              // coluna INT: envia número
  senha?: boolean;               // oculto, não aparece na lista, vazio ao editar = mantém
}
export interface Tema {
  titulo: string;
  icone: ComponentProps<typeof Ionicons>["name"];
  api: Api;
  campos: Campo[];
}

export default function TelaCadastro({ tema }: { tema: Tema }) {
  const { api, campos } = tema;
  const [lista, setLista] = useState<Registro[]>([]);
  const [carregando, setCarregando] = useState(false);
  const [aberto, setAberto] = useState(false);                 // modal visível?
  const [editandoId, setEditandoId] = useState<number | null>(null); // null = criando
  const [form, setForm] = useState<Record<string, string>>({});

  const carregar = useCallback(async () => {
    setCarregando(true);
    const r = await api.listar();
    setCarregando(false);
    if (r.ok) setLista(r.data ?? []);
    else Alert.alert("Erro ao listar", r.message);
  }, [api]);

  // roda toda vez que a tela GANHA FOCO (o Drawer mantém telas montadas; useEffect não bastaria)
  useFocusEffect(useCallback(() => { carregar(); }, [carregar]));

  function abrir(item?: Registro) {
    setEditandoId(item?.id ?? null);
    setForm(Object.fromEntries(
      campos.map((c) => [c.chave, c.senha || item?.[c.chave] == null ? "" : String(item[c.chave])]),
    ));
    setAberto(true);
  }

  async function salvar() {
    const dados: Registro = {};
    for (const c of campos) {
      const bruto = form[c.chave] ?? "";
      const v = c.senha ? bruto : bruto.trim();
      if (c.senha && !v && editandoId) continue;                   // editar sem digitar senha = não envia
      dados[c.chave] = c.numero ? (v === "" ? null : Number(v)) : v;
    }
    const r = editandoId ? await api.editar(editandoId, dados) : await api.criar(dados);
    if (!r.ok) return Alert.alert("Falha ao salvar", r.message);
    setAberto(false);
    carregar();
  }

  function excluir(item: Registro) {
    Alert.alert("Excluir", "Confirma a exclusão?", [
      { text: "Cancelar", style: "cancel" },
      { text: "Excluir", style: "destructive", onPress: async () => {
          const r = await api.excluir(item.id);
          if (!r.ok) return Alert.alert("Falha ao excluir", r.message);
          carregar();
      } },
    ]);
  }

  return (
    <View style={estilos.tela}>
      <View style={estilos.barraTopo}>
        <Text style={{ color: cores.suave }}>{lista.length} registro(s)</Text>
        <ButtonFatec
          titleButton="Novo"
          onFunctionButton={() => abrir()}
          icon={<Ionicons name="add-circle" size={22} color="#fff" />}
          styleButton={{ width: 110, height: 38, marginTop: 0, backgroundColor: cores.destaque }}
        />
      </View>

      <FlatList
        data={lista}
        keyExtractor={(i) => String(i.id)}
        refreshing={carregando}
        onRefresh={carregar}
        ItemSeparatorComponent={() => <View style={{ height: 10 }} />}
        ListEmptyComponent={<Text style={{ textAlign: "center", color: cores.suave }}>Nenhum registro.</Text>}
        renderItem={({ item }) => (
          <View style={estilos.card}>
            {campos.filter((c) => !c.senha).map((c) => (
              <View key={c.chave}>
                <Text style={estilos.cardRotulo}>{c.rotulo}</Text>
                <Text style={estilos.cardValor}>{String(item[c.chave] ?? "—")}</Text>
              </View>
            ))}
            <View style={estilos.acoes}>
              <ButtonFatec onFunctionButton={() => abrir(item)}
                styleButton={[estilos.botaoMini, { backgroundColor: cores.primaria }]}
                icon={<Ionicons name="create-outline" size={20} color="#fff" />} />
              <ButtonFatec onFunctionButton={() => excluir(item)}
                styleButton={[estilos.botaoMini, { backgroundColor: cores.perigo }]}
                icon={<Ionicons name="trash-outline" size={20} color="#fff" />} />
            </View>
          </View>
        )}
      />

      <Modal visible={aberto} animationType="slide" onRequestClose={() => setAberto(false)}>
        <ScrollView contentContainerStyle={estilos.telaAuth} keyboardShouldPersistTaps="handled">
          <Text style={estilos.titulo}>{editandoId ? "Editar" : "Novo"} · {tema.titulo}</Text>
          <View style={{ height: 24 }} />
          {campos.map((c) => (
            <CampoTexto
              key={c.chave}
              rotulo={c.senha && editandoId ? `${c.rotulo} (vazio = manter)` : c.rotulo}
              value={form[c.chave] ?? ""}
              onChangeText={(v) => setForm({ ...form, [c.chave]: v })}
              keyboardType={c.numero ? "numeric" : c.teclado}
              secureTextEntry={c.senha}
            />
          ))}
          <ButtonFatec titleButton="Salvar" onFunctionButton={salvar} />
          <ButtonFatec titleButton="Cancelar" onFunctionButton={() => setAberto(false)}
            styleButton={{ backgroundColor: cores.suave, marginTop: 12 }} />
        </ScrollView>
      </Modal>
    </View>
  );
}
🧠 Bloco a bloco

Tema e Campo são o "contrato" entre este componente e o temas.ts. Tema novo = preencher esse formato.
Criar × editar usa o mesmo formulário: editandoId === null → api.criar; com id → api.editar. Um estado só (aberto) controla o modal.
useFocusEffect recarrega a lista sempre que você volta à tela (ex.: depois de criar um estado e abrir cidades).
form só guarda texto (TextInput só devolve string); a conversão para número acontece uma vez, em salvar (c.numero). Vazio em coluna INT NULL vai como null (não 0).
Senha: não é trimada (espaço pode fazer parte); fica fora da lista; ao editar, se vier vazia, nem é enviada → a API mantém o hash antigo.
Object.fromEntries(campos.map(...)) monta {nome: "...", email: "..."} a partir da lista de campos, e é o que faz o formulário funcionar para qualquer tema.
FlatList (e não ScrollView + map) só renderiza o que aparece na tela; keyExtractor precisa de texto único (o id); refreshing/onRefresh dão o "puxar para atualizar".
Excluir sempre pede confirmação; depois recarrega a lista do servidor (a tela nunca "inventa" o estado).
♻ Os dois botões de ícone reusam estilos.botaoMini e só trocam a cor: é assim que se especializa o ButtonFatec sem criar outro componente.
PARTE C — O que você DIGITA na prova (app) ✍
C.1 src/app/_layout.tsx (RootLayout) — provider + porteiro único
import "react-native-gesture-handler";                  // obrigatório para o Drawer
import AuthProvider, { useAuth } from "@/utils/AuthContext";
import { Stack } from "expo-router";
import { GestureHandlerRootView } from "react-native-gesture-handler";

function Rotas() {
  const { isLoggedIn } = useAuth();
  return (
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Protected guard={isLoggedIn}>
        <Stack.Screen name="(auth)" />
      </Stack.Protected>
      <Stack.Protected guard={!isLoggedIn}>
        <Stack.Screen name="login" />
        <Stack.Screen name="register" />
      </Stack.Protected>
    </Stack>
  );
}

export default function RootLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <AuthProvider>
        <Rotas />
      </AuthProvider>
    </GestureHandlerRootView>
  );
}
🧠

Stack.Protected guard={...}: telas dentro do bloco só existem quando o guard é true. Logou → (auth) aparece e o app vai para lá; saiu → o roteador volta ao primeiro grupo permitido (login). É o único lugar que decide quem entra onde.
Rotas é um componente separado porque useAuth() só funciona dentro do AuthProvider; no RootLayout ele ainda estaria "do lado de fora".
Não retornamos null enquanto carrega: a splash (segurada no AuthContext) cobre a tela até isReady, e nesse mesmo instante o estado já está correto.
GestureHandlerRootView envolve tudo porque o Drawer usa gestos; sem ele a tela do Drawer fica em branco.
⚠ Se o seu expo-router for antigo e não tiver Stack.Protected, o plano B é <Redirect href="/login" /> dentro de (auth)/_layout.tsx quando !isLoggedIn, além de esperar isReady.
C.2 src/api/apiAuth.ts — traduz a API para o formato que o AuthContext espera
import { http } from "./apiConfig";

// { ok:true, token, user }  ou  { ok:false, message }
export async function AuthLogin(login: string, password: string) {
  const r = await http<{ token: string; user: { id: number; name: string } }>("/auth/login", {
    method: "POST",
    body: { login, password },
  });
  if (!r.ok) return { ok: false as const, message: r.message };
  return { ok: true as const, token: r.data.token, user: r.data.user };
}

export const AuthRegister = (name: string, login: string, password: string) =>
  http("/auth/register", { method: "POST", body: { name, login, password } });
🧠

as const deixa ok como true/false literal: o TypeScript entende que, depois de if (!r.ok) return, existem token e user.
Único arquivo a adaptar se a API do professor for diferente (ver seção E.2).
AuthRegister devolve a resposta padrão (ok, message); a tela decide o que mostrar.
C.3 src/api/apiUsers.ts
import { criarApi } from "./apiConfig";
export const apiUsers = criarApi("users");
C.4 src/config/temas.ts — a configuração de todos os cadastros
import { criarApi } from "@/api/apiConfig";
import { apiUsers } from "@/api/apiUsers";
import type { Tema } from "@/components/TelaCadastro";

export const temas = {
  clientes: {
    titulo: "Clientes", icone: "people", api: criarApi("clientes"),
    campos: [
      { chave: "nome", rotulo: "Nome" },
      { chave: "email", rotulo: "E-mail", teclado: "email-address" },
      { chave: "telefone", rotulo: "Telefone", teclado: "phone-pad" },
    ],
  },
  fornecedores: {
    titulo: "Fornecedores", icone: "business", api: criarApi("fornecedores"),
    campos: [
      { chave: "nome", rotulo: "Nome" },
      { chave: "cnpj", rotulo: "CNPJ", teclado: "numeric" },
      { chave: "telefone", rotulo: "Telefone", teclado: "phone-pad" },
      { chave: "email", rotulo: "E-mail", teclado: "email-address" },
    ],
  },
  estados: {
    titulo: "Estados", icone: "map", api: criarApi("estados"),
    campos: [
      { chave: "estado", rotulo: "Estado" },
      { chave: "uf", rotulo: "UF" },
      { chave: "regiao", rotulo: "Região" },
    ],
  },
  cidades: {
    titulo: "Cidades", icone: "location", api: criarApi("cidades"),
    campos: [
      { chave: "cidade", rotulo: "Cidade" },
      { chave: "uf", rotulo: "UF" },
      { chave: "populacao", rotulo: "População", numero: true },
    ],
  },
  roles: {
    titulo: "Roles", icone: "key", api: criarApi("roles"),
    campos: [{ chave: "role", rotulo: "Role" }],
  },
  users: {
    titulo: "Usuários", icone: "person-circle", api: apiUsers,
    campos: [
      { chave: "name", rotulo: "Nome" },
      { chave: "login", rotulo: "Login" },
      { chave: "password", rotulo: "Senha", senha: true },
    ],
  },
} satisfies Record<string, Tema>;
🧠

A chave do objeto (clientes, users...) tem 3 papéis: nome do endpoint, nome do arquivo da tela (clientes.tsx) e nome do item do Drawer. Por isso users (e não usuarios) → arquivo users.tsx.
chave de cada campo = nome da coluna em recursos.js. Se errar, a API ignora o campo (ou responde "Nenhum campo válido").
satisfies Record<string, Tema> confere o formato sem "achatar" os tipos; assim temas.clientes continua sendo reconhecido individualmente e icone só aceita nomes de ícone válidos.
♻ Cada bloco é o mesmo formato; o que muda é só título, ícone, endpoint e campos. numero: true → coluna INT; senha: true → campo de senha.
C.5 src/app/(auth)/(cadastros)/_layout.tsx (Menu Drawer)
import { temas } from "@/config/temas";
import { cores } from "@/styles/estilos";
import Ionicons from "@expo/vector-icons/Ionicons";
import { Drawer } from "expo-router/drawer";

export default function CadastrosLayout() {
  return (
    <Drawer
      screenOptions={{
        headerShown: true,
        headerStyle: { backgroundColor: cores.primaria },
        headerTintColor: cores.branco,
        drawerActiveTintColor: cores.primaria,
        drawerActiveBackgroundColor: cores.primariaClara,
      }}
    >
      {Object.entries(temas).map(([nome, t]) => (
        <Drawer.Screen
          key={nome}
          name={nome}
          options={{
            title: t.titulo,
            drawerIcon: ({ color, size }) => <Ionicons name={t.icone} size={size} color={color} />,
          }}
        />
      ))}
    </Drawer>
  );
}
🧠

♻ Em vez de 6 <Drawer.Screen> escritos à mão, o .map gera um por tema. Tema novo aparece no menu sozinho.
name tem que ser igual ao nome do arquivo (clientes → clientes.tsx), senão aparece o aviso "No route named ...".
O headerShown: true é o que mostra o botão ☰ que abre o menu; o título de cada tela vem de options.title.
Padrão de nomes: tab... nas abas, drawer... no Drawer.
C.6 As 6 telas de cadastro (♻ idênticas, só muda 1 palavra)
src/app/(auth)/(cadastros)/clientes.tsx:

import TelaCadastro from "@/components/TelaCadastro";
import { temas } from "@/config/temas";

export default function Clientes() {
  return <TelaCadastro tema={temas.clientes} />;
}
Arquivo	Troque temas.___ por	(nome da função é livre)
clientes.tsx	temas.clientes	Clientes
fornecedores.tsx	temas.fornecedores	Fornecedores
estados.tsx	temas.estados	Estados
cidades.tsx	temas.cidades	Cidades
roles.tsx	temas.roles	Roles
users.tsx	temas.users	Usuarios
🧠 Cada arquivo é obrigatório (o nome do arquivo vira a rota) e precisa de export default; a lógica toda está em TelaCadastro.

C.7 src/app/(auth)/_layout.tsx (Abas)
import { cores } from "@/styles/estilos";
import Ionicons from "@expo/vector-icons/Ionicons";
import { Tabs } from "expo-router";
import { ComponentProps } from "react";

const aba = (nome: ComponentProps<typeof Ionicons>["name"]) =>
  ({ color, size }: { color: string; size: number }) => <Ionicons name={nome} size={size} color={color} />;

export default function AuthLayout() {
  return (
    <Tabs
      screenOptions={{
        headerShown: false,
        tabBarActiveTintColor: cores.primaria,
        tabBarInactiveTintColor: cores.suave,
        tabBarStyle: { backgroundColor: cores.branco, borderTopColor: cores.borda, height: 62, paddingBottom: 8 },
      }}
    >
      <Tabs.Screen name="index" options={{ title: "Home", tabBarIcon: aba("home") }} />
      <Tabs.Screen name="perfil" options={{ title: "Perfil", tabBarIcon: aba("person") }} />
      <Tabs.Screen name="(cadastros)" options={{ title: "Cadastros", tabBarIcon: aba("albums") }} />
      <Tabs.Screen name="sair" options={{ title: "Sair", tabBarIcon: aba("log-out") }} />
    </Tabs>
  );
}
🧠

Não há verificação de login aqui: o porteiro está no _layout raiz (C.1); este arquivo só desenha as abas.
aba(nome) é uma função que devolve a função tabBarIcon (♻ evita repetir o <Ionicons ...> 4 vezes). O RN passa {color, size} e a cor muda sozinha quando a aba está ativa.
name="(cadastros)" refere-se à pasta: um navegador (Drawer) dentro de outro (Abas). O Drawer tem cabeçalho próprio, por isso aqui é headerShown: false (senão apareceriam dois cabeçalhos).
C.8 src/app/(auth)/index.tsx (Home) e perfil.tsx (PerfilUser) — ♻ mesmo desenho, dados diferentes
// index.tsx
import { useAuth } from "@/utils/AuthContext";
import { estilos } from "@/styles/estilos";
import { Text, View } from "react-native";

export default function Home() {
  const { user } = useAuth();
  return (
    <View style={estilos.telaCentro}>
      <Text style={estilos.titulo}>Olá, {user?.nome}!</Text>
      <Text style={estilos.subtitulo}>Use as abas para navegar e o menu “Cadastros” para gerenciar os dados.</Text>
    </View>
  );
}
// perfil.tsx  → mesmo import e mesmo useAuth(); muda só o retorno:
<View style={estilos.telaCentro}>
  <View style={[estilos.card, { width: "100%" }]}>
    <Text style={estilos.cardRotulo}>Nome</Text>  <Text style={estilos.cardValor}>{user?.nome}</Text>
    <Text style={estilos.cardRotulo}>Login</Text> <Text style={estilos.cardValor}>{user?.email}</Text>
    <Text style={estilos.cardRotulo}>ID</Text>    <Text style={estilos.cardValor}>{user?.id}</Text>
  </View>
</View>
🧠 user?.nome: user pode ser null (o ?. evita erro). Como só existe tela dentro de (auth) quando há login, na prática nunca é null aqui. Os dados vêm do contexto: nenhuma chamada à API.

C.9 src/app/(auth)/sair.tsx (SairApp)
import ButtonFatec from "@/components/ButtonFatec";
import { cores, estilos } from "@/styles/estilos";
import { useAuth } from "@/utils/AuthContext";
import Ionicons from "@expo/vector-icons/Ionicons";
import { Alert, Text, View } from "react-native";

export default function Sair() {
  const { logOut } = useAuth();

  function confirmar() {
    Alert.alert("Sair", "Deseja encerrar a sessão?", [
      { text: "Cancelar", style: "cancel" },
      { text: "Sair", style: "destructive", onPress: () => logOut() },
    ]);
  }

  return (
    <View style={estilos.telaCentro}>
      <Ionicons name="log-out-outline" size={64} color={cores.primaria} />
      <Text style={estilos.subtitulo}>Encerrar a sessão neste aparelho?</Text>
      <ButtonFatec titleButton="Sair da conta" onFunctionButton={confirmar}
        styleButton={{ backgroundColor: cores.perigo }} />
    </View>
  );
}
🧠 logOut() apaga token + storage; como isLoggedIn vira false, o Stack.Protected leva ao login sozinho (nenhum router aqui). O Alert com 2 botões é a "confirmação antes de sair".

C.10 src/app/login.tsx (Login)
import ButtonFatec from "@/components/ButtonFatec";
import CampoTexto from "@/components/CampoTexto";
import LogoApp from "@/components/LogoApp";
import { estilos } from "@/styles/estilos";
import { useAuth } from "@/utils/AuthContext";
import Ionicons from "@expo/vector-icons/Ionicons";
import { Link } from "expo-router";
import { useState } from "react";
import { Alert, ScrollView, Text } from "react-native";

export default function Login() {
  const { logIn } = useAuth();
  const [login, setLogin] = useState("");
  const [senha, setSenha] = useState("");
  const [enviando, setEnviando] = useState(false);

  async function entrar() {
    if (!login.trim() || !senha) return Alert.alert("Atenção", "Informe login e senha.");
    setEnviando(true);
    const erro = await logIn(login.trim(), senha);   // null = sucesso (o porteiro já troca de tela)
    setEnviando(false);
    if (erro) Alert.alert("Não foi possível entrar", erro);
  }

  return (
    <ScrollView contentContainerStyle={estilos.telaAuth} keyboardShouldPersistTaps="handled">
      <LogoApp />
      <Text style={estilos.titulo}>Login</Text>
      <Text style={estilos.subtitulo}>Bem-vindo de volta! Entre para continuar.</Text>

      <CampoTexto rotulo="Login" value={login} onChangeText={setLogin} />
      <CampoTexto rotulo="Senha" value={senha} onChangeText={setSenha} secureTextEntry maxLength={20} />

      <Text style={estilos.subtitulo}>
        Não tem conta? <Link href="/register" style={estilos.link}>Registre-se</Link>
      </Text>

      <ButtonFatec
        titleButton={enviando ? "Entrando..." : "Acessar"}
        onFunctionButton={entrar}
        disabled={enviando}
        icon={<Ionicons name="checkmark-circle" size={26} color="#fff" />}
      />
    </ScrollView>
  );
}
🧠

Estado por campo (useState) + onChangeText={setX} = o "input capturando valor" da prova.
enviando desabilita o botão: evita duplo toque disparando duas requisições.
ScrollView + keyboardShouldPersistTaps="handled": o teclado não cobre os campos e o primeiro toque no botão funciona mesmo com o teclado aberto.
secureTextEntry esconde a senha. <Link href="/register"> navega sem código.
.trim() no login evita o espaço que o corretor do teclado costuma inserir.
C.11 src/app/register.tsx (Register) — ♻ igual ao Login
Muda em relação ao login.tsx	Como fica
import extra	import { AuthRegister } from "@/api/apiAuth"; e import { useRouter } from "expo-router"; (no lugar de useAuth)
estado extra	const [nome, setNome] = useState(""); + const router = useRouter();
campo extra (1º)	<CampoTexto rotulo="Nome completo" value={nome} onChangeText={setNome} autoCapitalize="words" maxLength={60} />
textos	título Registro; link de volta <Link href="/login" style={estilos.link}>Voltar ao login</Link>
função do botão	registrar (abaixo) no lugar de entrar; botão titleButton="Registrar"
async function registrar() {
  if (!nome.trim() || !login.trim() || !senha) return Alert.alert("Atenção", "Preencha todos os campos.");
  setEnviando(true);
  const r = await AuthRegister(nome.trim(), login.trim(), senha);
  setEnviando(false);
  if (!r.ok) return Alert.alert("Falha no registro", r.message);
  Alert.alert("Pronto!", "Conta criada. Faça o login.");
  router.replace("/login");
}
🧠 Só navega ao login se a API respondeu com sucesso. router.replace (e não push) tira o registro do histórico: o botão "voltar" não retorna a ele. Login já cadastrado volta como 409 e a mensagem da API aparece no alerta.

PARTE D — Rodar e testar
# terminal 1
cd api && npm run dev                      # DB_RESET=false depois da 1ª vez
# terminal 2
cd <app-nome-do-app> && npx expo start -c  # -c limpa o cache (obrigatório após instalar pacote/mudar app.json)
Celular e PC na mesma rede Wi-Fi → Expo Go → QR Code.

Roteiro de teste (também serve de demonstração): Registrar → Login → Home → Cadastros (Drawer) → Criar / Editar / Excluir → Perfil → Sair → fechar e reabrir o app (deve continuar logado até sair).

PARTE E — Consulta rápida
E.1 Tema novo em 2 minutos (ex.: produtos)
#	Onde	O que fazer
1	api/src/recursos.js	novo bloco produtos: { colunas: { ... } }
2	api/.env	DB_RESET=true uma vez (cria a tabela) → volta para false
3	src/config/temas.ts	novo bloco produtos: { titulo, icone, api: criarApi("produtos"), campos: [...] }
4	(cadastros)/produtos.tsx	copie clientes.tsx e troque temas.clientes por temas.produtos
O menu Drawer e a rota da API aparecem sozinhos.

E.2 Se a API do professor for diferente
Só se mexe em apiAuth.ts (e nos chave de temas.ts). Como http devolve ok (pelo status HTTP), message, data e também body (JSON completo), basta ler do lugar certo. Exemplo com a API usada em aula, cujo login responde { message: "sucess", data: "<token>", user: { id, name } }:

export async function AuthLogin(login: string, password: string) {
  const r = await http("/auth/login", { method: "POST", body: { login, password } });
  if (!r.ok) return { ok: false as const, message: r.message };
  return { ok: true as const, token: r.body.data, user: r.body.user };   // ← só estas linhas mudam
}
Conferir também: porta (PORTA em apiConfig.ts), nome dos campos enviados (login/password/name) e se a listagem devolve data: [...] (a da aula devolve). Se a API não usar JWT, o http continua funcionando: o header Authorization só vai se existir token.

E.3 Erros comuns
Sintoma	Causa provável	Correção
"Sem conexão com a API."	API desligada, MySQL desligado, porta errada, ou firewall do Windows bloqueando o Node	npm run dev rodando? mesma porta? liberar o Node no firewall; testar http://IP:3500/test no navegador do celular
Só falha no celular	localhost aponta para o celular / túnel	preencher IP_MANUAL em apiConfig.ts
Dados somem sozinhos	DB_RESET=true + --watch	trocar para false
Nenhum campo válido no body	chave do campo ≠ coluna em recursos.js	conferir os nomes
Registro duplicado	login já existe (UNIQUE)	usar outro login
401 em tudo / "Sessão expirada"	token de 1 h venceu	o app já desloga sozinho; entre de novo
Tela branca ao abrir o Drawer	falta GestureHandlerRootView, o import "react-native-gesture-handler" ou @react-navigation/drawer	conferir C.1 e instalar o pacote
Cannot find module '@/...'	cache do Metro	npx expo start -c
Aviso "No route named X"	name do Tabs.Screen/Drawer.Screen ≠ nome do arquivo/pasta	igualar os nomes
Login OK mas volta ao login	falha ao gravar/ler o AsyncStorage	conferir expo install @react-native-async-storage/async-storage e reiniciar com -c
.env da API mudou e nada aconteceu	--watch não observa o .env	Ctrl+C e npm run dev
E.4 Ordem sugerida na prova (do mais valioso ao menos)
Pastas + .env + npm install → recursos.js → db.js → helpers.js → crud.js → auth.js → middleware.js → routes.js → server.js → rodar → DB_RESET=false
usersTest.http (registrar + login + rota protegida)
Levar os 📦 para as pastas → expo install (async-storage + drawer)
_layout.tsx raiz → apiAuth.ts → login.tsx → register.tsx → testar login de ponta a ponta
(auth)/_layout → Home → Perfil → Sair
apiUsers.ts → temas.ts → (cadastros)/_layout → as 6 telas (copiar/colar)
Commits: git add . && git commit -m "..." ao fechar cada bloco
PARTE F — Mapa de conexões (para explicar oralmente)
login.tsx ──logIn()──▶ AuthContext ──AuthLogin()──▶ apiAuth ──http()──▶ POST /auth/login
                            │  aplicar(): definirToken() + setSessao() + AsyncStorage[AUTH_KEY]
                            ▼
_layout raiz: Stack.Protected(guard = isLoggedIn) ──▶ vai para (auth) [ou volta ao login]

clientes.tsx ─▶ TelaCadastro(temas.clientes) ─▶ api.listar() ─▶ http(): põe "Bearer <token>" ─▶ GET /clientes
                                                                    │                              │
                                              401? ─▶ aoSessaoExpirar ─▶ logout      exigirToken ─▶ crud("clientes") ─▶ MySQL
Arquivo pronto 📦	Exige que exista
AuthContext.tsx	apiAuth.ts com AuthLogin · apiConfig.ts com AUTH_KEY, definirToken, aoSessaoExpirar
TelaCadastro.tsx	Api/Registro exportados por apiConfig.ts · objeto tema no formato Tema
temas.ts (✍)	criarApi · apiUsers.ts · tipo Tema do TelaCadastro
(cadastros)/_layout.tsx (✍)	um arquivo .tsx para cada chave de temas
CampoTexto, ButtonFatec, LogoApp	estilos.ts (cores, estilos) · assets/images/icon.png