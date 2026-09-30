# COLINHA — API + App (`prova_2ams_<nome-sobrenome>`)

**Como está organizada**
- **Seção 0**: suas anotações viradas em checklist.
- **Seção 1**: API do zero (Node + Express 5 + MySQL), cada arquivo com o *porquê* linha a linha.
- **Seção 2**: setup do app.
- **Seção 3**: arquivos que você **leva prontos** (contexto, estilos, componentes), cada um com "como usar" logo abaixo.
- **Seção 4**: o que você **escreve na hora** (rotas e telas).
- **Seção 5**: tema novo em 2 minutos, fluxo e erros comuns.

**Convenção:** onde um padrão se repete, aparece a marca **↻ Repete em: …** e o código não é escrito de novo, só a tabela do que muda.

---

# 0. Suas anotações (checklist)

| Item | Valor |
|---|---|
| Pasta do projeto | `prova_2ams_<nome-sobrenome>/` contendo `api/` e `<app-nome-do-app>/` |
| Banco | `<nome-do-banco-novo>` (a API cria sozinha se não existir) |
| Porta da API | `____` (a da prova). **Mesma porta** em: `.env` da API, `usersTest.http` e `apiConfig.ts` do app |
| `DB_RESET` | `true` na 1ª execução, **depois trocar para `false`** |
| Depois de configurar o `.env` | `cd api/` → `npm run dev` |
| Levar pronto | `AuthContext`, estilos (css das telas), componentes próprios (seção 3) |
| Expo | `npm run reset-project` (limpa o template) · tecla `r` no terminal do Expo recarrega o app |
| Storage | `npx expo install @react-native-async-storage/async-storage` |

```bash
mkdir prova_2ams_nome-sobrenome && cd prova_2ams_nome-sobrenome
mkdir api
npx create-expo-app@latest nome-do-app
```

**Por quê:** a API e o app são dois projetos independentes que conversam por HTTP; por isso duas pastas irmãs, cada uma com seu `package.json` e seu `node_modules`.

---

# 1. API

## 1.1 Projeto Node

```bash
cd api
npm init -y
npm install express mysql2 dotenv bcryptjs jsonwebtoken
mkdir -p src/database src/routes src/middlewares src/tests
```

Edite o `package.json` (troque/adicione estas chaves):

```json
"type": "module",
"scripts": { "dev": "node --watch src/server.js" }
```

Crie `api/.gitignore`:

```
node_modules/
.env
```

| Pacote / opção | Para que serve |
|---|---|
| `express` | servidor HTTP e rotas. A versão instalada hoje é a **5**, que já captura erros de funções `async` (na 4 seria preciso `try/catch` em cada rota) |
| `mysql2` | driver do MySQL. Usamos o subpacote `mysql2/promise` para poder usar `await` |
| `dotenv` | lê o arquivo `.env` e joga em `process.env` |
| `bcryptjs` | transforma a senha em **hash**; a senha real nunca fica no banco |
| `jsonwebtoken` | gera e valida o token de login (JWT) |
| `"type": "module"` | habilita `import/export` e `await` solto no topo do arquivo |
| `node --watch` | reinicia sozinho ao salvar (dispensa nodemon) |
| `.gitignore` | evita subir a senha do banco e a chave do JWT |

## 1.2 `api/.env`

```env
API_PORT=____
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=
DB_DATABASE=<nome-do-banco-novo>
DB_RESET=true
JWT_SECRET=troque-por-uma-frase-longa
```

**Por quê:** configuração fica fora do código. `DB_RESET=true` apaga e recria todas as tabelas **a cada vez que o servidor reinicia** (com `--watch`, isso é a cada salvamento!). Por isso, depois da primeira execução, mude para `false` ou seus dados somem.

## 1.3 `src/database/tabelas.js`, as tabelas em SQL puro

```js
const criar = (nome, colunas) => ({
  nome,
  sql: `CREATE TABLE IF NOT EXISTS ${nome} (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ${colunas},
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  )`,
});

export const tabelas = [
  criar("users", "name VARCHAR(100) NOT NULL, login VARCHAR(150) NOT NULL UNIQUE, password VARCHAR(255) NOT NULL"),
  criar("roles", "role VARCHAR(100) NOT NULL"),
  criar("clientes", "nome VARCHAR(100) NOT NULL, telefone VARCHAR(20), email VARCHAR(250)"),
  criar("fornecedores", "nome VARCHAR(100) NOT NULL, cnpj VARCHAR(20), telefone VARCHAR(20), email VARCHAR(250)"),
  criar("estados", "estado VARCHAR(80) NOT NULL, uf CHAR(2) NOT NULL, regiao VARCHAR(30)"),
  criar("cidades", "cidade VARCHAR(100) NOT NULL, uf CHAR(2) NOT NULL, populacao INT"),
];
```

**Por quê, linha a linha:**
- `criar(nome, colunas)` é uma função que monta o `CREATE TABLE`. Toda tabela precisa de `id` e `created_at`, então isso fica escrito **uma vez** aqui e você só informa as colunas de cada tema.
- `AUTO_INCREMENT PRIMARY KEY`: o banco gera o `id` sozinho.
- `NOT NULL` = obrigatório. Sem ele a coluna aceita vazio (ex.: `telefone`).
- `UNIQUE` em `login`: o próprio MySQL impede dois usuários com o mesmo login (o erro é tratado no `server.js`, passo 1.9).
- `IF NOT EXISTS`: não dá erro se a tabela já existir.
- **Tema novo = 1 linha `criar(...)` neste array** (seção 5.1).

## 1.4 `src/database/conexao.js`, conecta e cria banco + tabelas

```js
import "dotenv/config";
import mysql from "mysql2/promise";
import { tabelas } from "./tabelas.js";

const { DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_DATABASE, DB_RESET } = process.env;
const acesso = { host: DB_HOST, port: Number(DB_PORT) || 3306, user: DB_USER, password: DB_PASSWORD };

// 1) conexão "solta" (sem database) só para criar o banco, caso ainda não exista
const admin = await mysql.createConnection(acesso);
await admin.query(`CREATE DATABASE IF NOT EXISTS \`${DB_DATABASE}\``);
await admin.end();

// 2) pool já apontando para o banco; é o que o resto da API importa
export const db = mysql.createPool({ ...acesso, database: DB_DATABASE });

// 3) reset opcional e criação das tabelas
if (DB_RESET === "true") {
  for (const t of tabelas) await db.query(`DROP TABLE IF EXISTS ${t.nome}`);
}
for (const t of tabelas) await db.query(t.sql);
console.log("[db] banco e tabelas prontos");
```

**Por quê:**
1. Não dá para conectar em um banco que não existe; por isso a primeira conexão **não** informa `database`.
2. `pool` reaproveita conexões abertas (mais rápido que abrir uma por requisição).
3. `await` no topo do arquivo (permitido pelo `"type": "module"`) garante que tabelas existam **antes** de qualquer rota ser criada.
4. `Number(DB_PORT) || 3306`: o `.env` entrega texto; o driver quer número, e 3306 é o padrão se faltar.

## 1.5 `src/routes/criarCrud.js`, a "fábrica" de CRUD (escrita 1 vez, usada em todas as tabelas)

```js
import { Router } from "express";
import { db } from "../database/conexao.js";

const AUTOMATICAS = ["id", "created_at"];
const erro = (status, message) => Object.assign(new Error(message), { status });

export async function criarCrud(tabela, { ocultar = [], preparar = async (corpo) => corpo } = {}) {
  // colunas lidas do PRÓPRIO banco: dispensa listar campos duas vezes
  const [colunas] = await db.query(`SHOW COLUMNS FROM \`${tabela}\``);
  const permitidas = colunas.map((c) => c.Field).filter((c) => !AUTOMATICAS.includes(c));

  const limpar = (linha) =>
    Object.fromEntries(Object.entries(linha).filter(([chave]) => !ocultar.includes(chave)));

  async function extrair(corpo) {
    const dados = await preparar(corpo ?? {});
    const campos = Object.keys(dados).filter((c) => permitidas.includes(c));
    if (campos.length === 0) throw erro(400, "Nenhum campo válido enviado");
    return { campos, valores: campos.map((c) => dados[c]) };
  }

  const rota = Router();

  rota.get("/", async (req, res) => {
    const [linhas] = await db.query(`SELECT * FROM \`${tabela}\` ORDER BY id`);
    res.json({ data: linhas.map(limpar) });
  });

  rota.get("/:id", async (req, res) => {
    const [linhas] = await db.execute(`SELECT * FROM \`${tabela}\` WHERE id = ?`, [req.params.id]);
    if (linhas.length === 0) throw erro(404, "Registro não encontrado");
    res.json({ data: limpar(linhas[0]) });
  });

  rota.post("/", async (req, res) => {
    const { campos, valores } = await extrair(req.body);
    const marcas = campos.map(() => "?").join(", ");
    const [r] = await db.execute(`INSERT INTO \`${tabela}\` (${campos.join(", ")}) VALUES (${marcas})`, valores);
    res.status(201).json({ data: { id: r.insertId } });
  });

  rota.put("/:id", async (req, res) => {
    const { campos, valores } = await extrair(req.body);
    const sets = campos.map((c) => `${c} = ?`).join(", ");
    const [r] = await db.execute(`UPDATE \`${tabela}\` SET ${sets} WHERE id = ?`, [...valores, req.params.id]);
    if (r.affectedRows === 0) throw erro(404, "Registro não encontrado");
    res.json({ message: "Atualizado" });
  });

  rota.delete("/:id", async (req, res) => {
    const [r] = await db.execute(`DELETE FROM \`${tabela}\` WHERE id = ?`, [req.params.id]);
    if (r.affectedRows === 0) throw erro(404, "Registro não encontrado");
    res.json({ message: "Removido" });
  });

  return rota;
}
```

**Por quê, peça por peça:**
- **Uma função para todas as tabelas**: recebe o nome da tabela e devolve um `Router` com os 5 endpoints (`GET /`, `GET /:id`, `POST /`, `PUT /:id`, `DELETE /:id`). É isso que evita escrever model/controller/rota para cada tema.
- **`permitidas`**: o corpo da requisição vem do cliente e não é confiável. Só entram colunas que existem no banco. Os `?` protegem os *valores* contra SQL injection, mas os *nomes* de coluna vão direto na string do SQL; por isso o filtro `permitidas` é obrigatório. `id` e `created_at` ficam de fora para ninguém sobrescrevê-los.
- **`ocultar`**: lista de colunas que nunca saem na resposta (usado em `users` para esconder `password`).
- **`preparar`**: função opcional que transforma o corpo antes de gravar (usado em `users` para gerar o hash). O padrão devolve o corpo sem mexer.
- **`erro(status, message)`**: cria um erro com código HTTP; o tratador do `server.js` transforma em resposta JSON.
- **`throw` dentro de `async`**: no Express 5, vira resposta de erro automaticamente.
- **Formato de resposta**: sucesso → `{ data: ... }` ou `{ message }`; falha → `{ message }` com status 4xx. O app decide pelo **status HTTP** (`resp.ok`), não por texto.
- **`affectedRows === 0`**: o `id` não existe → 404.
- **`db.execute` vs `db.query`**: `execute` usa prepared statement (mais seguro para valores). `query` foi usado onde não há parâmetros vindos do cliente.

## 1.6 `src/middlewares/autenticar.js`, o "porteiro" do token

```js
import jwt from "jsonwebtoken";

export function autenticar(req, res, next) {
  const [tipo, token] = (req.headers.authorization ?? "").split(" ");
  if (tipo !== "Bearer" || !token) return res.status(401).json({ message: "Token não informado" });
  try {
    req.usuarioId = jwt.verify(token, process.env.JWT_SECRET).id;
    next();
  } catch {
    res.status(401).json({ message: "Token inválido ou expirado" });
  }
}
```

**Por quê:** o app manda `Authorization: Bearer <token>`. `split(" ")` separa a palavra `Bearer` do token. `jwt.verify` confere a assinatura com o `JWT_SECRET` e o prazo; se falhar, cai no `catch` (401). `next()` libera a requisição para a rota seguinte.

## 1.7 `src/routes/auth.js`, registro e login (públicas)

```js
import { Router } from "express";
import bcrypt from "bcryptjs";
import jwt from "jsonwebtoken";
import { db } from "../database/conexao.js";

const auth = Router();

auth.post("/register", async (req, res) => {
  const { name, login, password } = req.body ?? {};
  if (!name || !login || !password) {
    return res.status(400).json({ message: "Informe name, login e password" });
  }
  const hash = await bcrypt.hash(password, 10);
  await db.execute("INSERT INTO users (name, login, password) VALUES (?, ?, ?)", [name, login, hash]);
  res.status(201).json({ message: "Usuário criado" });
});

auth.post("/login", async (req, res) => {
  const { login, password } = req.body ?? {};
  const [[usuario]] = await db.execute("SELECT id, name, password FROM users WHERE login = ?", [login]);
  const confere = usuario && (await bcrypt.compare(password ?? "", usuario.password));
  if (!confere) return res.status(401).json({ message: "Login ou senha inválidos" });

  const token = jwt.sign({ id: usuario.id }, process.env.JWT_SECRET, { expiresIn: "8h" });
  res.json({ data: token, user: { id: usuario.id, name: usuario.name } });
});

export default auth;
```

**Por quê:**
- `req.body ?? {}`: no Express 5, sem corpo o `req.body` é `undefined`; o `?? {}` evita quebrar ao desestruturar.
- `bcrypt.hash(senha, 10)`: o `10` é o custo do algoritmo (mais alto = mais lento e mais seguro).
- **Login duplicado**: o `UNIQUE` do banco gera o erro `ER_DUP_ENTRY`, que o `server.js` converte em 409. Não precisa consultar antes.
- `const [[usuario]]`: `execute` devolve `[linhas, metadados]`; o segundo `[ ]` pega a primeira linha (fica `undefined` se não achar).
- **Mensagem única** ("Login ou senha inválidos") para os dois casos (usuário inexistente ou senha errada), para não revelar quais logins existem.
- **Resposta do login**: `{ data: <token>, user: { id, name } }`. É o contrato que o `AuthLogin` do app lê.

## 1.8 `src/routes/index.js`, junta tudo (aqui entra cada tema novo)

```js
import { Router } from "express";
import bcrypt from "bcryptjs";
import auth from "./auth.js";
import { criarCrud } from "./criarCrud.js";
import { autenticar } from "../middlewares/autenticar.js";

const rotas = Router();

rotas.use("/auth", auth);          // públicas: ficam ANTES do porteiro
rotas.use(autenticar);             // daqui para baixo tudo exige token

rotas.use("/users", await criarCrud("users", {
  ocultar: ["password"],
  preparar: async (corpo) =>
    corpo.password ? { ...corpo, password: await bcrypt.hash(corpo.password, 10) } : corpo,
}));

for (const tema of ["roles", "clientes", "fornecedores", "estados", "cidades"]) {
  rotas.use(`/${tema}`, await criarCrud(tema));
}

export default rotas;
```

**Por quê:**
- **A ordem manda**: `/auth` vem antes de `autenticar`, senão ninguém conseguiria fazer login.
- `users` usa a mesma fábrica, mas com duas opções: esconde o `password` nas respostas e gera o hash quando a senha é enviada (senha nunca é gravada em texto).
- Os demais temas usam a fábrica **sem opções**; o `for` cria as 5 rotas de uma vez. **Tema novo = adicionar o nome neste array.**

## 1.9 `src/server.js`

```js
import "dotenv/config";
import os from "node:os";
import express from "express";
import rotas from "./routes/index.js";

const porta = Number(process.env.API_PORT) || 3500;
const app = express();

app.use(express.json());
app.get("/test", (req, res) => res.json({ message: "API no ar", porta }));
app.use(rotas);

// rota inexistente
app.use((req, res) => res.status(404).json({ message: "Rota não encontrada" }));

// tratador de erros: TODO erro lançado nas rotas cai aqui
app.use((err, req, res, next) => {
  if (err.code === "ER_DUP_ENTRY") return res.status(409).json({ message: "Registro duplicado (ex.: login já existe)" });
  if (["ER_BAD_NULL_ERROR", "ER_NO_DEFAULT_FOR_FIELD"].includes(err.code)) {
    return res.status(400).json({ message: "Preencha os campos obrigatórios" });
  }
  if (err.code === "ER_DATA_TOO_LONG") return res.status(400).json({ message: "Algum texto passou do tamanho permitido" });
  if (err.status) return res.status(err.status).json({ message: err.message });
  console.error(err);
  res.status(500).json({ message: "Erro interno do servidor" });
});

const ip = Object.values(os.networkInterfaces()).flat()
  .find((i) => i?.family === "IPv4" && !i.internal)?.address ?? "localhost";
app.listen(porta, () => console.log(`API em http://${ip}:${porta}`));
```

**Por quê:**
- `express.json()` transforma o corpo em objeto JS; sem ele `req.body` não existe.
- `/test` é um "está vivo?" para conferir no navegador.
- Os **dois** `app.use` finais precisam vir **depois** das rotas: o primeiro pega URL inexistente; o segundo (4 parâmetros: `err, req, res, next`) é o tratador de erros. O Express só o reconhece como tratador por ter exatamente 4 parâmetros.
- Tradução de erros do MySQL para mensagens úteis: campo obrigatório vazio, texto grande demais, login repetido.
- O IP impresso é o que vai no `apiConfig.ts` do app. Se abrir no navegador do PC (Expo web) e der erro de CORS: `npm i cors` e `app.use(cors())` (celular/emulador não precisam).

Rode `npm run dev`; deve aparecer `[db] banco e tabelas prontos` e `API em http://...`. **Agora troque `DB_RESET=false`.**

## 1.10 `src/tests/usersTest.http` (extensão *REST Client* do VS Code)

```http
@porta = ____
@base = http://localhost:{{porta}}

### 1) registrar
POST {{base}}/auth/register
Content-Type: application/json

{ "name": "Aluno Teste", "login": "aluno", "password": "123456" }

### 2) login (o nome "login" guarda a resposta para reaproveitar)
# @name login
POST {{base}}/auth/login
Content-Type: application/json

{ "login": "aluno", "password": "123456" }

### 3) token vindo do login
@token = {{login.response.body.data}}

### 4) listar clientes (protegido)
GET {{base}}/clientes
Authorization: Bearer {{token}}

### 5) criar cliente
POST {{base}}/clientes
Authorization: Bearer {{token}}
Content-Type: application/json

{ "nome": "Maria", "email": "maria@fatec.com" }
```

**Por quê:** testa a API sozinha antes de tocar no app: se aqui falhar, o problema é da API. **Confira o valor de `@porta` na 1ª linha.** Sem token, os passos 4 e 5 devem responder 401.

---

# 2. App

## 2.1 Setup

```bash
cd ../nome-do-app
npm run reset-project
npx expo install @react-native-async-storage/async-storage @react-navigation/drawer react-native-gesture-handler react-native-reanimated
```

| Passo | Por quê |
|---|---|
| `reset-project` | apaga o template de exemplo e deixa uma pasta `app` vazia. Ele pergunta se move os exemplos para outra pasta; qualquer resposta serve |
| `async-storage` | guarda a sessão no aparelho (o login sobrevive ao fechar o app) |
| `drawer` + `gesture-handler` + `reanimated` | o `expo-router/drawer` depende dos três. Se já estiverem no template, o comando só confere as versões. Se o Metro reclamar de `react-native-worklets`, rode `npx expo install react-native-worklets` |
| `expo install` (e não `npm install`) | escolhe a versão compatível com o seu SDK do Expo |

**Pasta `src/app`:** se após o reset a pasta `app/` estiver na raiz, crie `src/` e mova `app/` para dentro (`src/app`). O Expo Router aceita as duas, mas a árvore da prova usa `src/`.

**Alias `@/`:** abra `tsconfig.json` e deixe assim (o padrão do template aponta para a raiz):

```json
"paths": { "@/*": ["./src/*"] }
```

Com isso `@/components/ButtonFatec` significa `src/components/ButtonFatec`. Reinicie com `npx expo start -c`.

## 2.2 Árvore final

```
src/
├── api/
│   ├── apiConfig.ts      URL da API + chave do storage
│   ├── http.ts           único fetch do app        (novo: evita repetir fetch)
│   ├── apiAuth.ts        AuthLogin / AuthRegister
│   ├── apiCrud.ts        criarApi(recurso)         (novo: 1 fábrica p/ todos os temas)
│   └── apiUsers.ts       criarApi("users")
├── components/           ButtonFatec  LogoApp  CampoTexto  TelaCrud       ← PRONTOS
├── context/              AuthContext.tsx                                  ← PRONTO
├── styles/               tema.ts  telas.ts                                ← PRONTOS
└── app/                              (tudo aqui dentro VIRA ROTA)
    ├── _layout.tsx                   RootLayout
    ├── login.tsx   register.tsx
    └── (auth)/                       grupo protegido (não aparece na URL)
        ├── _layout.tsx               guarda de rota + abas
        ├── index.tsx                 Home
        ├── perfil.tsx                PerfilUser
        ├── sair.tsx                  SairApp
        └── (cadastros)/              Menu Drawer
            ├── _layout.tsx
            └── clientes.tsx  fornecedores.tsx  estados.tsx  cidades.tsx  usuarios.tsx  roles.tsx
```

**Atenção:** `utils/` e `components/` dentro de `(auth)` (como no seu rascunho) **virariam rotas** e gerariam aviso de "sem default export". Mantenha só telas dentro de `app/`; o resto fica em `src/` ao lado.

---

# 3. Arquivos prontos (levar de casa)

Cada bloco traz o código e, logo abaixo, **como usar**.

## 3.1 API do app: `src/api/`

`apiConfig.ts`

```ts
export const apiUrl = "http://192.168.0.0:0000";   // IP do PC + porta da API
export const CHAVE_SESSAO = "sessao-fatec";
```

**Como usar:** é a **única** linha que você edita no dia (IP que a API imprimiu ao subir + porta). Emulador Android usa `10.0.2.2` no lugar do IP. **`localhost` no celular aponta para o próprio celular**, não para o PC. `CHAVE_SESSAO` é a "gaveta" do AsyncStorage; `AuthContext` e `http.ts` precisam usar a **mesma**.

`http.ts`

```ts
import AsyncStorage from "@react-native-async-storage/async-storage";
import { apiUrl, CHAVE_SESSAO } from "./apiConfig";

export interface Resposta {
  ok: boolean;      // true se o status HTTP foi 2xx
  status: number;   // 0 = não conseguiu falar com a API
  body: any;        // JSON devolvido pela API
}

export async function http(caminho: string, metodo = "GET", corpo?: object): Promise<Resposta> {
  try {
    const salvo = await AsyncStorage.getItem(CHAVE_SESSAO);
    const token: string | null = salvo ? JSON.parse(salvo).token : null;

    const resp = await fetch(apiUrl + caminho, {
      method: metodo,
      headers: {
        "Content-Type": "application/json",
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
      },
      body: corpo ? JSON.stringify(corpo) : undefined,
    });
    const body = await resp.json().catch(() => ({}));
    return { ok: resp.ok, status: resp.status, body };
  } catch {
    return { ok: false, status: 0, body: { message: "Sem conexão com a API. Confira o IP e a porta." } };
  }
}
```

**Por quê:**
- **É o único `fetch` do app.** Nenhuma tela lida com token nem com `try/catch`.
- O `fetch` **não dá erro em 400/401/500**, só quando a rede cai; por isso olhamos `resp.ok` (status 2xx).
- O `catch` devolve a mesma "forma" de resposta (`ok:false`), então a tela trata falha de rede e erro da API do mesmo jeito: `if (!r.ok) Alert.alert(r.body.message)`.
- `.json().catch(() => ({}))`: se a resposta vier sem JSON, não quebra.

`apiAuth.ts`

```ts
import { http } from "./http";

export async function AuthLogin(login: string, password: string) {
  const r = await http("/auth/login", "POST", { login, password });
  if (!r.ok) return { ok: false as const, mensagem: r.body?.message ?? "Falha no login" };
  return {
    ok: true as const,
    token: r.body.data as string,
    usuario: { id: r.body.user.id as number, nome: r.body.user.name as string, login },
  };
}

export async function AuthRegister(dados: { name: string; login: string; password: string }) {
  const r = await http("/auth/register", "POST", dados);
  return { ok: r.ok, mensagem: r.body?.message as string | undefined };
}
```

**Como usar:** `AuthLogin` é chamado **só** pelo `AuthContext`; a tela de registro chama `AuthRegister` direto. O `as const` em `ok` faz o TypeScript saber que, depois de `if (!r.ok)`, o resultado tem `token` e `usuario`.

`apiCrud.ts`

```ts
import { http } from "./http";

export type Dados = Record<string, any>;

export const criarApi = (recurso: string) => ({
  listar: () => http(`/${recurso}`),
  criar: (dados: Dados) => http(`/${recurso}`, "POST", dados),
  atualizar: (id: number, dados: Dados) => http(`/${recurso}/${id}`, "PUT", dados),
  excluir: (id: number) => http(`/${recurso}/${id}`, "DELETE"),
});

export type ApiCrud = ReturnType<typeof criarApi>;
```

`apiUsers.ts`

```ts
import { criarApi } from "./apiCrud";

export default criarApi("users");
```

**Como usar:** espelha o `criarCrud` da API. Em cada tela de tema: `const api = criarApi("clientes")` (o texto é o **mesmo prefixo de rota** registrado em `routes/index.js`). Usuários usa `apiUsers`, porque a rota da API é `/users` mas a tela chama `usuarios`.

## 3.2 Estilos: `src/styles/`

`tema.ts` (cores; mude aqui e o app inteiro muda)

```ts
export const tema = {
  primaria: "#0F766E",
  destaque: "#F59E0B",
  perigo: "#DC2626",
  neutro: "#64748B",
  fundo: "#F1F5F9",
  cartao: "#FFFFFF",
  texto: "#0F172A",
  textoSuave: "#64748B",
  borda: "#E2E8F0",
  campo: "#EEF2F6",
};
```

`telas.ts` (estilos das páginas)

```ts
import { StyleSheet } from "react-native";
import { tema } from "./tema";

export default StyleSheet.create({
  // telas de fora do login (login/register)
  authContainer: { flex: 1, alignItems: "center", padding: 24, paddingTop: 64, backgroundColor: tema.fundo },
  authTitulo: { fontSize: 30, fontWeight: "700", color: tema.primaria, marginTop: 20 },
  authSubtitulo: { color: tema.textoSuave, marginTop: 6, marginBottom: 24 },
  rodape: { marginTop: 18, color: tema.textoSuave },
  link: { color: tema.destaque, fontWeight: "700" },

  // telas internas
  tela: { flex: 1, padding: 16, backgroundColor: tema.fundo },
  telaCentro: { flex: 1, alignItems: "center", justifyContent: "center", padding: 24, backgroundColor: tema.fundo },
  tituloTela: { fontSize: 24, fontWeight: "700", color: tema.texto },
  cartao: {
    width: "100%", backgroundColor: tema.cartao, borderRadius: 14, padding: 16,
    borderWidth: 1, borderColor: tema.borda, marginTop: 12,
  },
  rotulo: { fontSize: 12, color: tema.textoSuave },
  valor: { fontSize: 16, color: tema.texto, marginBottom: 8 },
  avatar: { width: 96, height: 96, borderRadius: 48, backgroundColor: tema.primaria, alignItems: "center", justifyContent: "center" },
  avatarTexto: { color: "#fff", fontSize: 40, fontWeight: "700" },
});
```

**Como usar:** `import estilos from "@/styles/telas"` e `import { tema } from "@/styles/tema"`. Aplicação: `authContainer` na `<View>` raiz de login/register; `authTitulo`/`authSubtitulo` nos textos do topo; `rodape` + `link` na frase "Não tem conta? …"; `tela`/`telaCentro` nas telas internas; `cartao`/`rotulo`/`valor` nos blocos de dado (Perfil). Para ajustar sem editar o arquivo: `style={[estilos.cartao, { marginTop: 0 }]}` (array = mistura de estilos).

## 3.3 Componentes: `src/components/`

`ButtonFatec.tsx`

```tsx
import { ReactNode } from "react";
import { StyleProp, StyleSheet, Text, TouchableOpacity, ViewStyle } from "react-native";
import { tema } from "@/styles/tema";

type Variante = "primario" | "perigo" | "neutro";

interface Props {
  titleButton?: string;
  onFunctionButton?: () => void;
  icon?: ReactNode;
  variant?: Variante;
  small?: boolean;
  disabled?: boolean;
  styleButton?: StyleProp<ViewStyle>;
}

const fundo: Record<Variante, string> = { primario: tema.primaria, perigo: tema.perigo, neutro: tema.neutro };

export default function ButtonFatec({
  titleButton, onFunctionButton, icon, variant = "primario", small = false, disabled = false, styleButton,
}: Props) {
  return (
    <TouchableOpacity
      onPress={onFunctionButton}
      disabled={disabled}
      activeOpacity={0.8}
      style={[s.base, small ? s.pequeno : s.grande, { backgroundColor: fundo[variant] }, disabled && s.apagado, styleButton]}
    >
      {icon}
      {titleButton ? <Text style={[s.texto, icon ? { marginLeft: 8 } : null]}>{titleButton}</Text> : null}
    </TouchableOpacity>
  );
}

const s = StyleSheet.create({
  base: { flexDirection: "row", alignItems: "center", justifyContent: "center", borderRadius: 12 },
  grande: { width: "100%", height: 48, marginTop: 20 },
  pequeno: { paddingHorizontal: 14, height: 38 },
  texto: { color: "#fff", fontSize: 16, fontWeight: "600" },
  apagado: { opacity: 0.5 },
});
```

**Como usar:**

| Prop | Obrigatória | Efeito |
|---|---|---|
| `titleButton` | não | texto (sem ele, só ícone) |
| `onFunctionButton` | não | função ao tocar. **Sem argumento:** `onFunctionButton={salvar}`. **Com argumento:** `onFunctionButton={() => excluir(item)}` (sem a seta a função executa na hora da renderização) |
| `icon` | não | qualquer elemento: `icon={<Ionicons name="add" size={20} color="#fff" />}` |
| `variant` | não | `"primario"` (verde-petróleo, padrão) · `"perigo"` (vermelho) · `"neutro"` (cinza) |
| `small` | não | botão compacto, largura pelo conteúdo (para lado a lado e dentro de cards). Sem ele, ocupa 100% da largura |
| `disabled` | não | bloqueia o toque e deixa meio transparente (use enquanto envia) |
| `styleButton` | não | sobrescreve qualquer estilo do botão |

`LogoApp.tsx`

```tsx
import Ionicons from "@expo/vector-icons/Ionicons";
import { StyleSheet, View } from "react-native";
import { tema } from "@/styles/tema";

export default function LogoApp() {
  return (
    <View style={s.circulo}>
      <Ionicons name="school" size={56} color="#fff" />
    </View>
  );
}

const s = StyleSheet.create({
  circulo: { width: 110, height: 110, borderRadius: 55, backgroundColor: tema.primaria, alignItems: "center", justifyContent: "center" },
});
```

**Como usar:** `<LogoApp />`, sem props, no topo de login e register. Usa um ícone em vez de imagem para não depender de caminho de arquivo em `assets/`. Para trocar o desenho, mude o `name` do `Ionicons`.

`CampoTexto.tsx`

```tsx
import { StyleSheet, Text, TextInput, TextInputProps, View } from "react-native";
import { tema } from "@/styles/tema";

interface Props extends TextInputProps {
  label: string;
}

export default function CampoTexto({ label, style, ...resto }: Props) {
  return (
    <View style={s.grupo}>
      <Text style={s.label}>{label}</Text>
      <TextInput
        style={[s.input, style]}
        placeholderTextColor={tema.textoSuave}
        autoCapitalize="none"
        {...resto}
      />
    </View>
  );
}

const s = StyleSheet.create({
  grupo: { width: "100%", marginBottom: 14 },
  label: { fontSize: 12, fontWeight: "600", color: tema.textoSuave, marginBottom: 4 },
  input: { backgroundColor: tema.campo, borderRadius: 10, padding: 12, fontSize: 16, color: tema.texto },
});
```

**Como usar:** é um rótulo + `TextInput` num pacote só. `extends TextInputProps` significa que **qualquer** prop do `TextInput` funciona direto: `value`, `onChangeText`, `secureTextEntry`, `keyboardType`, `placeholder`, `maxLength`. `...resto` repassa o que sobrar, e como vem por último, você pode sobrescrever o `autoCapitalize="none"` padrão (ex.: `autoCapitalize="words"` para nome).

`TelaCrud.tsx`, lista + formulário + editar + excluir, em qualquer tema

```tsx
import Ionicons from "@expo/vector-icons/Ionicons";
import { useCallback, useEffect, useState } from "react";
import { Alert, FlatList, KeyboardTypeOptions, Pressable, StyleSheet, Text, View } from "react-native";
import type { ApiCrud, Dados } from "@/api/apiCrud";
import { tema } from "@/styles/tema";
import ButtonFatec from "./ButtonFatec";
import CampoTexto from "./CampoTexto";

export interface Campo {
  name: string;               // nome EXATO da coluna no banco
  label: string;              // texto mostrado na tela
  keyboard?: KeyboardTypeOptions;
  numeric?: boolean;          // coluna INT
  secure?: boolean;           // senha
}

interface Props {
  title: string;
  api: ApiCrud;
  fields: Campo[];
}

export default function TelaCrud({ title, api, fields }: Props) {
  const [itens, setItens] = useState<Dados[]>([]);
  const [carregando, setCarregando] = useState(false);
  const [formAberto, setFormAberto] = useState(false);
  const [editandoId, setEditandoId] = useState<number | null>(null);
  const [form, setForm] = useState<Record<string, string>>({});

  const carregar = useCallback(async () => {
    setCarregando(true);
    const r = await api.listar();
    setCarregando(false);
    if (r.ok) setItens(r.body.data);
    else Alert.alert("Erro ao listar", r.body?.message ?? "");
  }, [api]);

  useEffect(() => { carregar(); }, [carregar]);

  function abrirNovo() {
    setEditandoId(null);
    setForm({});
    setFormAberto(true);
  }

  function abrirEdicao(item: Dados) {
    const valores: Record<string, string> = {};
    for (const f of fields) valores[f.name] = f.secure ? "" : String(item[f.name] ?? "");
    setEditandoId(item.id);
    setForm(valores);
    setFormAberto(true);
  }

  function fechar() {
    setFormAberto(false);
    setEditandoId(null);
    setForm({});
  }

  async function salvar() {
    const dados: Dados = {};
    for (const f of fields) {
      const bruto = form[f.name] ?? "";
      const texto = f.secure ? bruto : bruto.trim();
      if (f.secure && texto === "" && editandoId !== null) continue;   // senha vazia ao editar = mantém a antiga
      dados[f.name] = f.numeric ? (texto === "" ? null : Number(texto)) : texto;
    }
    const r = editandoId === null ? await api.criar(dados) : await api.atualizar(editandoId, dados);
    if (!r.ok) return Alert.alert("Não foi possível salvar", r.body?.message ?? "");
    fechar();
    carregar();
  }

  function excluir(item: Dados) {
    Alert.alert("Excluir", "Confirma a exclusão deste registro?", [
      { text: "Cancelar", style: "cancel" },
      {
        text: "Excluir",
        style: "destructive",
        onPress: async () => {
          const r = await api.excluir(item.id);
          if (!r.ok) return Alert.alert("Não foi possível excluir", r.body?.message ?? "");
          carregar();
        },
      },
    ]);
  }

  return (
    <View style={s.tela}>
      <View style={s.topo}>
        <Text style={s.titulo}>{title}</Text>
        <ButtonFatec small titleButton="Novo" onFunctionButton={abrirNovo}
          icon={<Ionicons name="add" size={20} color="#fff" />} />
      </View>

      {formAberto && (
        <View style={s.form}>
          <Text style={s.formTitulo}>{editandoId === null ? "Novo registro" : `Editando #${editandoId}`}</Text>
          {fields.map((f) => (
            <CampoTexto
              key={f.name}
              label={f.label}
              value={form[f.name] ?? ""}
              onChangeText={(v) => setForm((atual) => ({ ...atual, [f.name]: v }))}
              secureTextEntry={f.secure}
              keyboardType={f.numeric ? "numeric" : f.keyboard}
            />
          ))}
          <View style={s.linha}>
            <ButtonFatec small titleButton="Salvar" onFunctionButton={salvar} />
            <ButtonFatec small variant="neutro" titleButton="Cancelar" onFunctionButton={fechar} />
          </View>
        </View>
      )}

      <FlatList
        style={{ marginTop: 8 }}
        data={itens}
        keyExtractor={(i) => String(i.id)}
        refreshing={carregando}
        onRefresh={carregar}
        ItemSeparatorComponent={() => <View style={{ height: 8 }} />}
        ListEmptyComponent={<Text style={s.vazio}>Nenhum registro.</Text>}
        renderItem={({ item }) => (
          <Pressable style={s.card} onPress={() => abrirEdicao(item)}>
            <View style={{ flex: 1 }}>
              {fields.filter((f) => !f.secure).map((f) => (
                <Text key={f.name} style={s.linhaTexto}>
                  <Text style={s.rotulo}>{f.label}: </Text>
                  {String(item[f.name] ?? "-")}
                </Text>
              ))}
            </View>
            <ButtonFatec small variant="perigo" onFunctionButton={() => excluir(item)}
              icon={<Ionicons name="trash" size={18} color="#fff" />} />
          </Pressable>
        )}
      />
    </View>
  );
}

const s = StyleSheet.create({
  tela: { flex: 1, padding: 16, backgroundColor: tema.fundo },
  topo: { flexDirection: "row", alignItems: "center", justifyContent: "space-between", marginBottom: 8 },
  titulo: { fontSize: 20, fontWeight: "700", color: tema.texto },
  form: { backgroundColor: tema.cartao, borderRadius: 14, padding: 14, borderWidth: 1, borderColor: tema.borda },
  formTitulo: { fontWeight: "700", color: tema.primaria, marginBottom: 10 },
  linha: { flexDirection: "row", gap: 10 },
  card: { flexDirection: "row", alignItems: "center", gap: 10, backgroundColor: tema.cartao, borderRadius: 12, padding: 12, borderWidth: 1, borderColor: tema.borda },
  linhaTexto: { color: tema.texto, marginBottom: 2 },
  rotulo: { fontWeight: "700", color: tema.textoSuave },
  vazio: { textAlign: "center", color: tema.textoSuave, marginTop: 24 },
});
```

**Como usar:** cada tela de tema é só `<TelaCrud title="…" api={api} fields={[…]} />` (seção 4.6). Ela já faz sozinha:

| O que faz | Como |
|---|---|
| Carregar ao abrir | `useEffect` chama `carregar` |
| Puxar para atualizar | `refreshing` + `onRefresh` do `FlatList` |
| Criar / editar | o formulário é o mesmo; `editandoId` vazio → `criar`, preenchido → `atualizar` |
| Abrir edição | tocar no card |
| Excluir | botão vermelho + confirmação; recarrega a lista depois |
| Erro | `Alert` com a mensagem que a API mandou (`r.body.message`) |

Detalhes que importam:
- `fields[].name` **precisa ser igual ao nome da coluna** no `tabelas.js`; senão a API ignora o campo ou responde "Nenhum campo válido".
- `numeric: true` converte o texto para número antes de enviar (colunas `INT`); vazio vira `null`.
- `secure: true`: o campo esconde o texto, **não aparece na lista** e, se ficar vazio ao editar, **não é enviado** (a senha antiga permanece).
- `useCallback(..., [api])`: o `api` precisa ser criado **fora** do componente da tela (nível do arquivo), senão seria um objeto novo a cada renderização e a lista recarregaria em loop.
- O formulário do estado usa `setForm((atual) => ({ ...atual, [f.name]: v }))`: copia os valores atuais e troca só o campo digitado (`[f.name]` = chave dinâmica).

## 3.4 `src/context/AuthContext.tsx`

```tsx
import AsyncStorage from "@react-native-async-storage/async-storage";
import { SplashScreen } from "expo-router";
import { createContext, PropsWithChildren, useContext, useEffect, useState } from "react";
import { AuthLogin } from "@/api/apiAuth";
import { CHAVE_SESSAO } from "@/api/apiConfig";

SplashScreen.preventAutoHideAsync();

export interface Usuario {
  id: number;
  nome: string;
  login: string;
}

interface Sessao {
  usuario: Usuario;
  token: string;
}

interface AuthCtx {
  usuario: Usuario | null;
  estaLogado: boolean;
  carregando: boolean;
  entrar: (login: string, senha: string) => Promise<{ ok: boolean; mensagem?: string }>;
  sair: () => Promise<void>;
}

const AuthContext = createContext<AuthCtx>({} as AuthCtx);
export const useAuth = () => useContext(AuthContext);

export default function AuthProvider({ children }: PropsWithChildren) {
  const [usuario, setUsuario] = useState<Usuario | null>(null);
  const [carregando, setCarregando] = useState(true);

  // ao abrir o app: tenta recuperar a sessão salva
  useEffect(() => {
    AsyncStorage.getItem(CHAVE_SESSAO)
      .then((salvo) => {
        if (salvo) setUsuario((JSON.parse(salvo) as Sessao).usuario);
      })
      .catch(() => {})
      .finally(() => setCarregando(false));
  }, []);

  // só some com a splash quando terminou de ler o storage
  useEffect(() => {
    if (!carregando) SplashScreen.hideAsync();
  }, [carregando]);

  async function entrar(login: string, senha: string) {
    const r = await AuthLogin(login, senha);
    if (!r.ok) return { ok: false, mensagem: r.mensagem };
    await AsyncStorage.setItem(CHAVE_SESSAO, JSON.stringify({ usuario: r.usuario, token: r.token }));
    setUsuario(r.usuario);
    return { ok: true };
  }

  async function sair() {
    await AsyncStorage.removeItem(CHAVE_SESSAO);
    setUsuario(null);
  }

  return (
    <AuthContext.Provider value={{ usuario, estaLogado: usuario !== null, carregando, entrar, sair }}>
      {children}
    </AuthContext.Provider>
  );
}
```

**Por quê, peça por peça:**
- **Contexto** = "variável global" que qualquer tela lê sem passar props. O `AuthProvider` envolve o app (4.1) e o hook `useAuth()` lê o contexto (evita escrever `useContext(AuthContext)` em toda tela).
- **`carregando`** começa `true`: lê o AsyncStorage é assíncrono; sem esse estado, o app decidiria "não logado" por engano e piscaria a tela de login.
- **Splash**: `preventAutoHideAsync()` segura a tela de abertura; `hideAsync()` libera quando `carregando` vira `false`.
- **`entrar`**: chama a API; se falhar devolve `{ ok:false, mensagem }` para a tela mostrar; se der certo, grava `{ usuario, token }` no storage (é dele que o `http.ts` tira o token) e atualiza o estado.
- **`sair`**: apaga a sessão e zera o usuário. **Não navega**: quem redireciona é a guarda do `(auth)/_layout` (4.2), que reage ao `estaLogado` virar `false`.
- **`estaLogado`** é derivado (`usuario !== null`): não precisa de um segundo estado que possa ficar dessincronizado.

**Como usar (em qualquer tela):**

```tsx
const { usuario, estaLogado, carregando, entrar, sair } = useAuth();
```

| Campo | Uso |
|---|---|
| `usuario?.nome` / `login` / `id` | mostrar dados do usuário |
| `estaLogado`, `carregando` | guarda de rota (só o `(auth)/_layout` usa) |
| `entrar(login, senha)` | devolve `{ ok, mensagem }`. **Não navega**: a tela de login faz `router.replace("/")` se `ok` |
| `sair()` | limpa a sessão; a guarda leva sozinha para `/login` |

---

# 4. O que você escreve na hora

## 4.1 Raízes e telas de acesso

`src/app/_layout.tsx` (RootLayout)

```tsx
import { Stack } from "expo-router";
import { GestureHandlerRootView } from "react-native-gesture-handler";
import AuthProvider from "@/context/AuthContext";

export default function RootLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <AuthProvider>
        <Stack screenOptions={{ headerShown: false }} />
      </AuthProvider>
    </GestureHandlerRootView>
  );
}
```

**Por quê:** o layout raiz envolve **todas** as rotas. `AuthProvider` por fora para todas as telas enxergarem o login. `GestureHandlerRootView` é obrigatório para o Drawer (sem ele, tela em branco). `Stack` empilha `login`, `register` e o grupo `(auth)`; `headerShown: false` porque essas telas têm visual próprio.

`src/app/login.tsx`

```tsx
import Ionicons from "@expo/vector-icons/Ionicons";
import { Link, useRouter } from "expo-router";
import { useState } from "react";
import { Alert, Text, View } from "react-native";
import ButtonFatec from "@/components/ButtonFatec";
import CampoTexto from "@/components/CampoTexto";
import LogoApp from "@/components/LogoApp";
import { useAuth } from "@/context/AuthContext";
import estilos from "@/styles/telas";

export default function Login() {
  const [login, setLogin] = useState("");
  const [senha, setSenha] = useState("");
  const [enviando, setEnviando] = useState(false);
  const { entrar } = useAuth();
  const router = useRouter();

  async function acessar() {
    if (!login.trim() || !senha) return Alert.alert("Atenção", "Informe login e senha.");
    setEnviando(true);
    const r = await entrar(login.trim(), senha);
    setEnviando(false);
    if (r.ok) router.replace("/");
    else Alert.alert("Não foi possível entrar", r.mensagem);
  }

  return (
    <View style={estilos.authContainer}>
      <LogoApp />
      <Text style={estilos.authTitulo}>Login</Text>
      <Text style={estilos.authSubtitulo}>Bem-vindo de volta!</Text>

      <CampoTexto label="Login" value={login} onChangeText={setLogin} placeholder="seu login" />
      <CampoTexto label="Senha" value={senha} onChangeText={setSenha} secureTextEntry placeholder="sua senha" />

      <ButtonFatec titleButton="Acessar" onFunctionButton={acessar} disabled={enviando}
        icon={<Ionicons name="log-in" size={22} color="#fff" />} />

      <Text style={estilos.rodape}>
        Não tem conta? <Link href="/register" style={estilos.link}>Registre-se</Link>
      </Text>
    </View>
  );
}
```

**Por quê:**
- Validação local primeiro (campo vazio nem chega na API).
- `enviando` desabilita o botão durante a chamada (evita duplo toque).
- **`router.replace("/")`** e não `push`: troca a tela no histórico, então o botão "voltar" não retorna ao login. Aqui o login é quem navega porque ele mora **fora** do grupo protegido.
- `<Link href="/register">` navega sem código; `href` é o caminho da rota.
- `useState` guarda o que foi digitado; `onChangeText={setLogin}` é atalho de `(t) => setLogin(t)`.

`src/app/register.tsx`: **↻ Repete a estrutura de `login.tsx`.** Só muda o seguinte:

| Parte | No `register.tsx` |
|---|---|
| Imports | acrescente `import { AuthRegister } from "@/api/apiAuth";` e retire `useAuth` |
| Estados | `nome`, `login`, `senha` (sem `enviando`, se quiser) |
| Campos | `CampoTexto` **Nome** (`autoCapitalize="words"`), **Login**, **Senha** (`secureTextEntry`) |
| Título | "Registro" / "Crie sua conta" |
| Rodapé | "Já tem conta? `<Link href="/login">` Voltar" |
| Botão | `titleButton="Registrar"` chamando a função abaixo |

```tsx
async function registrar() {
  if (!nome.trim() || !login.trim() || !senha) return Alert.alert("Atenção", "Preencha todos os campos.");
  const r = await AuthRegister({ name: nome.trim(), login: login.trim(), password: senha });
  if (!r.ok) return Alert.alert("Não foi possível registrar", r.mensagem);
  Alert.alert("Pronto", "Conta criada! Faça o login.");
  router.replace("/login");
}
```

**Por quê:** só navega ao login **se** a API aceitou (`r.ok`). Login repetido volta com mensagem "Registro duplicado" (409) e cai no `Alert`. O objeto `{ name, login, password }` usa os nomes que a API espera (`nome` local vira `name`).

## 4.2 Grupo protegido: `src/app/(auth)/_layout.tsx`

```tsx
import Ionicons from "@expo/vector-icons/Ionicons";
import { Redirect, Tabs } from "expo-router";
import { useAuth } from "@/context/AuthContext";
import { tema } from "@/styles/tema";

type Icone = keyof typeof Ionicons.glyphMap;
const aba = (nome: Icone) => ({ color, size }: { color: string; size: number }) => (
  <Ionicons name={nome} size={size} color={color} />
);

export default function AuthLayout() {
  const { estaLogado, carregando } = useAuth();

  if (carregando) return null;                                   // ainda lendo o storage
  if (!estaLogado) return <Redirect href="/login" />;            // guarda de rota

  return (
    <Tabs
      screenOptions={{
        headerStyle: { backgroundColor: tema.primaria },
        headerTintColor: "#fff",
        tabBarActiveTintColor: tema.primaria,
      }}
    >
      <Tabs.Screen name="index" options={{ title: "Home", tabBarIcon: aba("home") }} />
      <Tabs.Screen name="perfil" options={{ title: "Perfil", tabBarIcon: aba("person") }} />
      <Tabs.Screen name="(cadastros)" options={{ title: "Cadastros", headerShown: false, tabBarIcon: aba("albums") }} />
      <Tabs.Screen name="sair" options={{ title: "Sair", tabBarIcon: aba("log-out") }} />
    </Tabs>
  );
}
```

**Por quê:**
1. **Protege todas as telas do grupo num lugar só**: sem login → `<Redirect>` para `/login`. E como reage ao `estaLogado`, o `sair()` também leva ao login sem código extra.
2. `if (carregando) return null` evita mandar para o login quem já tem sessão salva.
3. `Tabs.Screen name="..."` precisa ser **igual ao nome do arquivo/pasta**: `index`, `perfil`, `(cadastros)`, `sair`.
4. `headerShown: false` na aba `(cadastros)`: o Drawer dentro dela já traz cabeçalho próprio; sem isso apareceriam dois.
5. `aba(nome)` é uma função que devolve o `tabBarIcon`; escrita uma vez, usada nas 4 abas.

## 4.3 Telas simples do grupo

`(auth)/index.tsx` (Home)

```tsx
import { Text, View } from "react-native";
import { useAuth } from "@/context/AuthContext";
import estilos from "@/styles/telas";
import { tema } from "@/styles/tema";

export default function Home() {
  const { usuario } = useAuth();
  return (
    <View style={estilos.telaCentro}>
      <Text style={estilos.tituloTela}>Olá, {usuario?.nome}!</Text>
      <Text style={{ color: tema.textoSuave, marginTop: 6 }}>Abra a aba Cadastros para gerenciar os dados.</Text>
    </View>
  );
}
```

`(auth)/perfil.tsx` (PerfilUser), **↻ mesma casca da Home** (imports e `useAuth`); muda só o conteúdo:

```tsx
<View style={estilos.telaCentro}>
  <View style={estilos.avatar}>
    <Text style={estilos.avatarTexto}>{usuario?.nome.charAt(0).toUpperCase()}</Text>
  </View>
  <View style={estilos.cartao}>
    <Text style={estilos.rotulo}>Nome</Text><Text style={estilos.valor}>{usuario?.nome}</Text>
    <Text style={estilos.rotulo}>Login</Text><Text style={estilos.valor}>{usuario?.login}</Text>
    <Text style={estilos.rotulo}>ID</Text><Text style={estilos.valor}>{usuario?.id}</Text>
  </View>
</View>
```

**Por quê:** o avatar é a inicial do nome (sem foto). `?.` (encadeamento opcional) evita erro caso `usuario` seja `null` por um instante.

`(auth)/sair.tsx` (SairApp)

```tsx
import { Text, View } from "react-native";
import ButtonFatec from "@/components/ButtonFatec";
import { useAuth } from "@/context/AuthContext";
import estilos from "@/styles/telas";
import { tema } from "@/styles/tema";

export default function SairApp() {
  const { sair, usuario } = useAuth();
  return (
    <View style={estilos.telaCentro}>
      <Text style={estilos.tituloTela}>Deseja sair, {usuario?.nome}?</Text>
      <Text style={{ color: tema.textoSuave, marginTop: 6 }}>Você precisará fazer login novamente.</Text>
      <ButtonFatec titleButton="Sair da conta" variant="perigo" onFunctionButton={sair} />
    </View>
  );
}
```

**Por quê:** a aba "Sair" abre uma tela de confirmação de verdade (mais simples e previsível que interceptar o toque da aba). Ao tocar, `sair()` zera o usuário e a guarda do `(auth)/_layout` redireciona sozinha para `/login`, sem `router` aqui.

## 4.4 Menu Drawer: `(auth)/(cadastros)/_layout.tsx`

```tsx
import Ionicons from "@expo/vector-icons/Ionicons";
import { Drawer } from "expo-router/drawer";
import { tema } from "@/styles/tema";

const temas = [
  { name: "clientes", title: "Clientes", icon: "people" },
  { name: "fornecedores", title: "Fornecedores", icon: "business" },
  { name: "estados", title: "Estados", icon: "map" },
  { name: "cidades", title: "Cidades", icon: "location" },
  { name: "usuarios", title: "Usuários", icon: "person-circle" },
  { name: "roles", title: "Roles", icon: "key" },
] as const;

export default function CadastrosLayout() {
  return (
    <Drawer
      screenOptions={{
        headerStyle: { backgroundColor: tema.primaria },
        headerTintColor: "#fff",
        drawerActiveTintColor: tema.primaria,
      }}
    >
      {temas.map((t) => (
        <Drawer.Screen
          key={t.name}
          name={t.name}
          options={{
            title: t.title,
            drawerIcon: ({ color, size }) => <Ionicons name={t.icon} size={size} color={color} />,
          }}
        />
      ))}
    </Drawer>
  );
}
```

**Por quê:** os itens do menu vêm de uma **lista**, e não de 6 blocos iguais: tema novo = 1 objeto no array. `name` precisa ser igual ao **nome do arquivo** da tela. `as const` faz o TypeScript aceitar o texto do ícone como nome válido do Ionicons.

## 4.5 Como o app "sabe" quem pode entrar (resumo do fluxo)

```
abrir o app ─▶ AuthProvider lê AsyncStorage ─▶ carregando=false
                    │ tem sessão ─▶ usuario preenchido ─▶ (auth) libera as abas
                    │ sem sessão ─▶ (auth)/_layout ─▶ <Redirect "/login">
login.tsx ─ entrar() ─▶ AuthLogin ─▶ http ─▶ POST /auth/login
     ok ─▶ grava {usuario, token} ─▶ router.replace("/") ─▶ (auth)/index
telas de tema ─▶ criarApi ─▶ http (lê token do AsyncStorage) ─▶ GET /clientes ...
sair() ─▶ apaga sessão ─▶ usuario=null ─▶ guarda redireciona ─▶ /login
```

## 4.6 Telas de tema (`(auth)/(cadastros)/`)

`clientes.tsx`, o exemplo completo

```tsx
import { criarApi } from "@/api/apiCrud";
import TelaCrud from "@/components/TelaCrud";

const api = criarApi("clientes");                     // 1) prefixo da rota da API (fora do componente!)

export default function Clientes() {
  return (
    <TelaCrud
      title="Clientes"                                // 2) título
      api={api}
      fields={[                                       // 3) campos (name = coluna do banco)
        { name: "nome", label: "Nome" },
        { name: "email", label: "E-mail", keyboard: "email-address" },
        { name: "telefone", label: "Telefone", keyboard: "phone-pad" },
      ]}
    />
  );
}
```

**↻ Repete em:** `fornecedores.tsx`, `estados.tsx`, `cidades.tsx`, `roles.tsx`, `usuarios.tsx`. Mesma estrutura; **só mudam 3 coisas**:

| Arquivo | `criarApi(...)` | `title` | `fields` (`name` → `label`) | Extras |
|---|---|---|---|---|
| `fornecedores.tsx` | `"fornecedores"` | Fornecedores | `nome`→Nome · `cnpj`→CNPJ · `telefone`→Telefone · `email`→E-mail | `phone-pad` no telefone |
| `estados.tsx` | `"estados"` | Estados | `estado`→Estado · `uf`→UF · `regiao`→Região | |
| `cidades.tsx` | `"cidades"` | Cidades | `cidade`→Cidade · `uf`→UF · `populacao`→População | `numeric: true` em `populacao` |
| `roles.tsx` | `"roles"` | Roles | `role`→Role | |
| `usuarios.tsx` | **não usa `criarApi`**: `import api from "@/api/apiUsers"` | Usuários | `name`→Nome · `login`→Login · `password`→Senha | `secure: true` em `password` |

**Por quê:** cada arquivo só declara *qual endpoint* e *quais campos*; todo o CRUD está no `TelaCrud` (3.3). O `name` de cada campo é a coluna do banco; o `label` é só o texto na tela. Em `usuarios`, o `secure` garante que a senha não aparece na lista e que, na edição, senha vazia mantém a atual.

---

# 5. Rodar, tema novo e erros

## 5.1 Rodar

```bash
cd api && npm run dev                    # DB_RESET=false depois da 1ª vez
cd <app-nome-do-app> && npx expo start -c   # -c limpa o cache; depois, tecla r recarrega
```

Ordem do teste: registrar → login → Home → Cadastros → CRUD de um tema → Perfil → Sair.

## 5.2 Tema novo (uns 2 minutos)

| Onde | O que fazer |
|---   |---           |
| API `database/tabelas.js` | 1 linha `criar("novo", "coluna VARCHAR(100) NOT NULL, ...")` |
| API `routes/index.js` | acrescentar `"novo"` no array do `for` |
| API `.env` | `DB_RESET=true` **uma vez** para criar a tabela (depois `false`) |
| App | copiar `clientes.tsx` → `novo.tsx` (endpoint, título, campos) |
| App `(cadastros)/_layout.tsx` | 1 objeto no array `temas` |

Se a tabela nova precisar aparecer **sem** perder os dados das outras, dá para criá-la sem reset: com `DB_RESET=false`, o `CREATE TABLE IF NOT EXISTS` já cria só a tabela que ainda não existe.

## 5.3 Erros comuns

| Sintoma | Causa provável |
|---|---|
| "Sem conexão com a API" | IP/porta errados no `apiConfig.ts`; API desligada; celular em outra rede; `localhost` no celular |
| 401 em tudo (Home ok, listas não) | token expirou (8 h): **Sair** e entrar de novo |
| Dados somem a cada salvamento | `DB_RESET` ainda `true` |
| "Nenhum campo válido enviado" | `name` do campo no `fields` ≠ nome da coluna no `tabelas.js` |
| "Preencha os campos obrigatórios" | campo `NOT NULL` da tabela ficou vazio no formulário |
| "Registro duplicado" | login já existe (409) |
| Tela em branco ao abrir o Drawer | falta `GestureHandlerRootView` no RootLayout ou pacote do drawer não instalado |
| Aviso "missing default export" | arquivo que não é tela dentro de `src/app/` |
| Import `@/...` não resolve | `tsconfig.json` sem `"@/*": ["./src/*"]`, ou não reiniciou com `-c` |
| Lista em loop recarregando | `criarApi(...)` dentro do componente em vez de fora |
| Botão dispara na hora de renderizar | `onFunctionButton={fn(x)}` em vez de `onFunctionButton={() => fn(x)}` |