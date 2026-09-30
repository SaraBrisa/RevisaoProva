nome pasta pjt: prova_2ams_<nome-sobrenome>
[api, <app-nome-do-app>]
nome banco: <nome-do-banco-novo>
porta api:



.env
DB_HOST = localhost
DB_USER = root
DB_DATABASE = <nome-do-banco-novo>
DB_RESET = true e depois de rodar muda para false


após configurar o .env
cd api/
npm run dev


usersTest.http
verificar se a porta esta correta




trazer pronto:
arquivo de authcontext 
estilização(css das paginas)
componentes proprios

rotas das pastas
###
*tabela de sigla utilizadas na arvore organização na app*
| sigla | descricao
|--------------------|------
|<nome pasta>/ nome da pasta e sua indicacao
|(pasta)/ - nome da pasta com indicacao de agrupamento de rotas
|<nome> - nome arquivo tsx
|(nome componentes) - nome do componente visual (tsx)
src/
|     api/
|    |      | <apiConfig.ts>
|    |      | <apiAuth.ts>
|    |      | <apiUsers.ts>
|    |      | components
|    |      |       | <ButtonFatec.tsx>(ButtonFatec)
|    |      |       | <LogoApp.tsx>(LogoApp)
|     app/
|    |   (auth)/
|    |     |     (cadastros)
|    |     |          |   _layout.tsx
|    |     |          |   <tela do tema [cliente, fornecedor, estados, cidades]>
|    |     |          |   <tela usuarios>
|    |     |          |   <tela roles>
|    |     |          |   ...
|    |     |     <index.tsx>(Home)
|    |     |     <perfil.tsx> (PerfilUser)
|    |     |     <(cadastros)> [Menu Drawer]
|    |     |     <sair> (SairApp)
|    |     |     utils/
|    |     |     components
|    |     |     
|    |     <_layout.tsx>(RootLayout)
|    |     <login.tsx>(login)
|    |     <register.tsx>(register)
###

npm run reset
R - reseta
npx expo install @react-native-async-storage/async-storage