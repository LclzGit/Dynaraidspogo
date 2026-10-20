# Pokégo Raids: app pessoal de Max Battles

Documentação do projeto para continuar no Claude Code. Está em português, como o app.

> **Neste repositório** só estão `pokego.html` (na raiz, não em `app/`) e este arquivo. Os demais arquivos da tabela da seção 2 ficaram fora do repositório.

## 1. O que o app faz

App de uma página só (`app/pokego.html`), sem servidor, que abre com duplo clique no navegador.

- Mostra os counters de cada raid Dinamax/Gigamax/Eternamax, vindos do GO Hub.
- Você cadastra os seus Pokémon: nível, Ataque Max (1–3), Defesa Max (0–3), Cura Max (0–3) e IVs (0–15, opcional).
- Para cada raid, estima o dano/PS/cura **dos seus** Pokémon, ranqueia e sugere times de 3.
- Tem botão PT/EN (só interface; nomes de Pokémon e golpes ficam em português, do GO Hub).
- Backup em JSON (download, ou mostrar/copiar e colar de volta).
- Dados salvos em `localStorage` (chave `pokego-meu-time-v2`) só neste navegador.

## 2. Arquivos

| Arquivo | Para que serve |
|---|---|
| `app/pokego.html` | **App final**, com os dados embutidos. Esse é o que o usuário abre. |
| `app/pokego.bak.html` | Backup de uma versão anterior. |
| `app/pokego.py` | Versão antiga em tkinter (Python). Não é mais o foco. |
| `app/todos.json` | Dados das raids (158 entradas) usados pelo Python. |
| `app/meu_time_importar.json` | Time de exemplo do usuário, para importar. |
| `counters/todos.json` | Dataset unificado (fonte de verdade dos dados das raids). |
| `counters/lote-01..11.json` | Lotes brutos de coleta por páginas do GO Hub. |
| `data.json` | Dados originais da Eternatus (formato antigo). |
| `index.html` | Versão web antiga (depende da pasta `counters/`). Substituída pelo `pokego.html`. |

## 3. Formato dos dados das raids

Cada entrada em `DADOS.entradas`:

```json
{
  "alvo": "Azelf Dinamax",
  "url": "https://db.pokemongohub.net/pt/pokemon/482-Dynamax/counters-max-battles",
  "data_evento": "14/11",            // opcional
  "observacao": "...",               // opcional
  "atacantes": [{"nome": "...", "ataque_rapido": "...", "ataque_max": "...", "dano": 1208}],
  "tanques":   [{"nome": "...", "ataque_rapido": "...", "ps_perdidos_por_golpe": 11.6}],
  "curadores": [{"nome": "...", "ps_curados_por_uso": 68}]
}
```

- `dano` do atacante: número da página, sem unidade. Assume tudo no máximo.
- `ps_perdidos_por_golpe` do tanque: % do PS por golpe, menor é melhor.
- `ps_curados_por_uso` do curador: PS curado por uso da Cura Max nível 3.

## 4. Como o GO Hub calcula (base do que o app replica)

Pela metodologia do próprio site:
- **Atacantes:** dano de um Ataque Max, com nível do Pokémon, nível do Ataque Max e clima selecionados. Os números salvos são do padrão da página, que o usuário considera "tudo maximizado".
- **Tanques:** média de % de PS perdido por golpe, assumindo **IV perfeito (15/15/15)**, sem escudos.
- **Curadores:** Cura Max nível 3 = **16% do PS efetivo do curador**, limitado ao PS que falta.

## 5. Mecânicas de Max Battle (pesquisa)

- Potência do Ataque Max Dinamax: nível 1 = 250, nível 2 = 300, nível 3 = 350.
- Gigamax: 350 / 400 / 450.
- Barreira Max (Defesa Max): escudos, não aumenta o stat de defesa. O ranking de tanques do GO Hub ignora barreiras.
- Cura Max: nível 1 = 8%, nível 2 = 12%, nível 3 = 16% do PS máximo do curador.
- Zacian, Zamazenta e Eternatus não podem Dinamax.

## 6. Fórmulas usadas no app

### 6.1 Stats e CP
- Ataque efetivo = (ataque base + IV de ataque) × CPM(nível)
- PS efetivo = (PS base + IV de PS) × CPM(nível)
- Defesa efetiva = (defesa base + IV de defesa) × CPM(nível)
- CP máximo (nível 50, IV 15) = (A+15) × √(D+15) × √(S+15) × CPM50² / 10

Usado para **conferir** os status base (todos batem com o CP publicado, erro ≤ 0,06%).

### 6.2 CPM (multiplicador de nível)
Valores usados (níveis que o app consulta, de 20 a 50 em meio nível, na tabela `CPM_TAB`):

| Nível | CPM | Nível | CPM |
|---|---|---|---|
| 20 | 0,5974 | 36 | 0,76740 |
| 22 | 0,62657 | 40 | 0,79030 |
| 26,5 | 0,68768 | 47 | 0,82530 |
| 28 | 0,70688 | 50 | 0,84030 |
| 29,5 | 0,72558 | | |
| 31,5 | 0,74079 | | |
| 35 | 0,76156 | | |

Fonte: tabela de CPM do PokémonGO Hub. A tabela completa está em `CPM_TAB` no `pokego.html`.

### 6.3 Atacante
```
dano_seu = dano_pagina × [ (ataque_base + IV_atk_seu) × CPM(nivel_seu) ]
                        / [ (ataque_base + IV_ref)    × CPM(50) ]
                        × [ potencia(Max_seu) / 350 ]
```
- Referência: nível 50, Ataque Max 3, **IV_ref = 15** (escolha do app; o GO Hub não informa o IV do atacante).
- Potência: 250 / 300 / 350 para Max 1 / 2 / 3.
- Essa fórmula é uma **razão** com o número da página. O tipo do golpe e o STAB já estão no número da página, por isso não precisam ser recalculados para o mesmo Pokémon contra o mesmo alvo.

Ganho de subir o Ataque Max: 250→300 = +20%, 300→350 = +16,7%.

### 6.4 Tanque
```
pct_seu = pct_pagina × [ (PS_base+15)·(DEF_base+15)·CPM50² ] / [ (PS_base+IV_hp_seu)·(DEF_base+IV_def_seu)·CPM(nivel_seu)² ]
```
- Referência: nível 50, IV 15/15 (como o GO Hub, que assume IV perfeito para tanques).
- IV em branco conta como 0.
- Defesa Max não entra.

### 6.5 Curador
```
ps_seu = ps_pagina × [ (PS_base+IV_hp_seu) · CPM(nivel_seu) · pct_cura_seu ] / [ (PS_base+15) · CPM50 · 16 ]
```
- `pct_cura_seu` = 0 / 8 / 12 / 16 conforme Cura Max 0 / 1 / 2 / 3.
- Cura Max 0 → não cura → fica fora do ranking.
- Não modela o limite de "PS que falta".

### 6.6 Times de 3 Pokémon
Três opções, sempre sem repetir o mesmo Pokémon:
1. **2 tanques + 1 DPS**: DPS = melhor atacante seu; tanques = os dois melhores tanques que sobram.
2. **1 tanque + 1 curador + 1 DPS**: melhor tanque e melhor curador diferentes entre si.
3. **Tanque-curador + 2 DPS**: um Pokémon que está entre seus tanques e curadores (ex.: Blissey) ocupa um slot só.

## 7. Status base usados (validados pelo CP)

Forma padrão da espécie, exceto onde indicado.

| Pokémon | Ataque | Defesa | PS | Observação |
|---|---|---|---|---|
| Abra | 195 | 82 | 93 | |
| Alakazam | 271 | 167 | 146 | |
| Articuno | 192 | 236 | 207 | |
| Azelf | 270 | 151 | 181 | |
| Blissey | 129 | 169 | 496 | |
| Butterfree | 167 | 137 | 155 | |
| Centiskorch | 220 | 158 | 225 | |
| Chansey | 60 | 128 | 487 | |
| Charizard | 223 | 173 | 186 | |
| Cinderace | 238 | 163 | 190 | |
| Corviknight | 163 | 192 | 221 | |
| Cryogonal | 190 | 218 | 190 | |
| Darmanitan | 263 | 114 | 233 | |
| Dialga | — | — | — | não usado como atacante |
| Drampa | 231 | 164 | 186 | |
| Duraludon | 239 | 185 | 172 | |
| Electivire | 249 | 163 | 181 | |
| Eternatus | 278 | 192 | 268 | |
| Espeon | 261 | 175 | 163 | |
| Excadrill | 255 | 129 | 242 | |
| Flareon | 246 | 179 | 163 | |
| Flygon | 205 | 168 | 190 | |
| Gallade | 237 | 195 | 169 | igual à Gardevoir; conferir |
| Gardevoir | 237 | 195 | 169 | igual à Gallade; conferir |
| Gengar | 261 | 149 | 155 | forma comum, não Gigamax |
| Gigalith | 226 | 201 | 198 | |
| Glaceon | 238 | 205 | 163 | |
| Grimmsnarl | 227 | 139 | 216 | |
| Gyarados | 237 | 186 | 216 | |
| Hatterene | 237 | 182 | 149 | |
| Ho-Oh | 239 | 244 | 214 | |
| Hydreigon | 256 | 188 | 211 | |
| Inteleon | 262 | 142 | 172 | |
| Jolteon | 232 | 182 | 163 | |
| Kabutops | 220 | 186 | 155 | |
| Kadabra | 232 | 117 | 120 | |
| Kingler | 240 | 181 | 146 | |
| Lapras | 165 | 174 | 277 | |
| Latias | 228 | 246 | 190 | |
| Latios | 268 | 212 | 190 | |
| Leafeon | 216 | 219 | 163 | |
| Lugia | 193 | 310 | 235 | |
| Machamp | 234 | 159 | 207 | |
| Mesprit | 212 | 212 | 190 | |
| Metagross | 257 | 228 | 190 | |
| Moltres | 251 | 181 | 207 | |
| Omastar | 207 | 201 | 172 | |
| Raikou | 241 | 195 | 207 | |
| Regirock | 179 | 309 | 190 | |
| Registeel | 143 | 285 | 190 | |
| Rhydon | 222 | 171 | 233 | |
| Rhyperior | 241 | 190 | 251 | |
| Rillaboom | 239 | 168 | 225 | |
| Snorlax | 190 | 169 | 330 | |
| Suicune | 180 | 235 | 225 | |
| Sylveon | 203 | 205 | 216 | |
| Toxtricity | 224 | 140 | 181 | forma Amped; Low Key não conferida |
| Tsareena | 222 | 195 | 176 | |
| Umbreon | 126 | 240 | 216 | |
| Uxie | 156 | 270 | 181 | |
| Vaporeon | 205 | 161 | 277 | |
| Venusaur | 198 | 189 | 190 | |
| Wailmer | 136 | 68 | 277 | |
| Wailord | 175 | 87 | 347 | |
| Weavile | 243 | 171 | 172 | |
| Zacian (Espada Coroada) | 332 | 240 | 192 | confirmado pelo CP 5.629 |
| Zamazenta (Escudo Coroado) | 250 | 292 | 192 | confirmado pelo CP 4.717 |
| Zapdos | 253 | 185 | 207 | |

Sem status base no app (não estimados): Urshifu (892 indisponível no GO Hub), Toxtricity Low Key, Dialga/Palkia/Azelf como alvo.

## 8. Decisões tomadas (e o porquê)

- **Dado do site = referência máxima.** O app compara com a página assumindo nível 50, Ataque Max 3.
- **IV dos atacantes = 15 na referência.** Assumido; o GO Hub não diz. Se o site usar outro valor, trocar `REF.ivAtk`.
- **IV em branco = 0 no cadastro.** Subestima um pouco; a tela avisa.
- **Defesa Max não reduz o % do tanque** (é escudo).
- **Dano do Gengar calibrado** com K = 0,00766 para alvo Psíquico (só para comparação, não é o dano exato).
- Coleta de dados do GO Hub: a página não mostra fórmula nem valores padrão das configurações.

## 9. Como rodar / publicar

- Abrir `app/pokego.html` no navegador (duplo clique). Não precisa de servidor.
- Para regenerar o app a partir dos dados: o HTML tem o JSON embutido em `const DADOS = {...}`. Os status base estão em `const BASE`, a tabela de CPM em `const CPM_TAB`, e a referência em `const REF`.
- Testes feitos: `node --check` na parte de script; conferência do CP; testes isolados das funções de estimativa. Teste de fumaça em Chromium headless com tela de celular (390 px): abre sem erros no console e sem rolagem horizontal. Ainda não testado em celular real.

## 10. Pendências

1. Urshifu (892): aguardar o GO Hub publicar a página, ou usar outra fonte.
2. Toxtricity Low Key: status não conferido.
3. Gallade e Gardevoir: vieram iguais; conferir na fonte.
4. Gengar Gigamax: o app usa a forma comum.
5. Confirmar o IV de referência dos atacantes no GO Hub (hoje 15).
6. Modelar o limite da cura pelo PS que falta.
7. Testar no navegador e no celular real.
8. Chansey: confirmar a ordem dos status (60/128/487) em uma segunda fonte. O CP bate com essa ordem.
9. Os valores de dano da página ignoram o nível do usuário; o app só estima a partir da razão com a referência.

## 11. Fontes

- GO Hub, páginas de counters Max Battles e status por espécie: https://db.pokemongohub.net/pt/pokemon/
- Tabela de CPM: https://pokemongohub.net/post/article/pokemon-go-cpm-list/
- Status e CP do Zacian Espada Coroada: https://www.sportskeeda.com/pokemon/pokemon-go-crowned-sword-zacian-best-moveset-counters-is-good
- Status e CP do Zamazenta Escudo Coroado: https://www.dittobase.com/pokemon-go/pokedex/zamazenta-crowned-shield
