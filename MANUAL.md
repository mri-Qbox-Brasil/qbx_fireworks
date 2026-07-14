# qbx_fireworks — Manual

Fogos de artifício usáveis como item do inventário, mais shows coordenados disparados por comando ou export.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Permissões (ACE)](#permissões-ace)
4. [Configuração](#configuração)
5. [Fogos disponíveis](#fogos-disponíveis)
6. [Shows](#shows)
7. [Comandos](#comandos)
8. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
9. [Localização](#localização)
10. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qbx_core` | Sim | `CreateUseableItem`, notificações |
| `ox_lib` | Sim | Callbacks, `progressBar`, `requestNamedPtfxAsset`, `addCommand`, locale, `versionCheck` |
| `ox_inventory` | Sim | `Search` (isqueiro) e `RemoveItem` (consumo do fogo) |

---

## Instalação

1. Copie a pasta `qbx_fireworks` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure qbx_fireworks
   ```
3. Cadastre os itens no `ox_inventory` (`data/items.lua`). Os nomes precisam bater com os `itemName` do `config/shared.lua` — por padrão `firework1`, `firework2`, `firework3` e `firework4`.
4. Se `needLighter` estiver `true` (padrão), o item `lighter` também precisa existir no `ox_inventory`.

Não há SQL nem conflitos conhecidos com outros recursos.

---

## Permissões (ACE)

O comando `/startshow` é registrado com `restricted = 'group.admin'` no `lib.addCommand`. Para liberar, o grupo precisa existir no ACE:

```
add_ace group.admin command allow
```

---

## Configuração

Arquivo: `config/shared.lua` (retorna uma tabela Lua).

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `detonationTime` | number | Sim | Segundos entre a colocação do fogo e a detonação. Também usado como atraso inicial de cada sequência de show. Padrão: `5` |
| `needLighter` | bool | Sim | Se `true`, o jogador precisa ter o item `lighter` no inventário para acender o fogo |
| `fireworks` | table | Sim | Mapa `assetName -> { itemName, particleList }`. A chave é o nome do asset PTFX do GTA V |
| `fireworks[asset].itemName` | string | Sim | Nome do item no `ox_inventory` que dispara este fogo |
| `fireworks[asset].particleList` | string[] | Sim | Efeitos de partícula do asset sorteados aleatoriamente a cada explosão |
| `shows` | table | Não | Mapa `nomeDoShow -> { fireworks = {...} }`. O nome da chave é o argumento aceito por `/startshow` |
| `shows[nome].fireworks[i].asset` | string | Sim | Asset do fogo. Precisa existir em `fireworks` |
| `shows[nome].fireworks[i].coords` | vec3 | Sim | Ponto de lançamento |
| `shows[nome].fireworks[i].height` | number | Sim | Altura (em Z, a partir de `coords`) onde a explosão acontece |
| `shows[nome].fireworks[i].wait` | number | Sim | Atraso em ms antes de disparar o próximo item da sequência |

---

## Fogos disponíveis

Os quatro assets configurados de fábrica:

| Asset | Item | Nº de partículas |
|---|---|---|
| `proj_indep_firework` | `firework1` | 4 |
| `proj_indep_firework_v2` | `firework2` | 6 |
| `proj_xmas_firework` | `firework3` | 5 |
| `scr_indep_fireworks` | `firework4` | 8 |

Para adicionar um fogo novo, inclua uma entrada em `fireworks` com um asset PTFX válido do GTA V, a lista de partículas dele e o nome de um item existente no `ox_inventory`. O servidor registra o item como usável automaticamente na inicialização.

### Fluxo de uso do item

1. O jogador usa o item de fogo no inventário.
2. Se `needLighter` estiver ativo, o servidor verifica o item `lighter`; sem ele, o uso é abortado com notificação.
3. Barra de progresso de 3 segundos com a animação `anim@mp_fireworks / place_firework_3_box` e o prop `ind_prop_firework_03`.
4. O objeto do fogo é criado pelo servidor à frente do jogador; a statebag `qbx_fireworks:initiate` faz o dono da entidade assentá-lo no chão e congelá-lo.
5. Após `detonationTime` segundos, entre 10 e 15 explosões com partículas sorteadas são disparadas.
6. O item é removido do inventário. O objeto é deletado `detonationTime * 1000 + 26000` ms após a criação.

Se o jogador cancelar a barra de progresso, o item **não** é consumido.

---

## Shows

Um show é uma sequência pré-definida de fogos disparada de uma vez para todos os jogadores do servidor. O único show que vem configurado é `lsia` (aeroporto de Los Santos), com 29 sequências.

O servidor percorre cada item da sequência: cria o prop no chão, sorteia de 10 a 15 partículas do asset, envia o evento para todos os clientes e espera `wait` ms antes do próximo.

Para criar um show novo, adicione uma chave em `shows`:

```lua
shows = {
    ['ano_novo'] = {
        fireworks = {
            { asset = 'proj_xmas_firework', coords = vec3(100.0, 200.0, 30.0), height = 50.0, wait = 4000 },
            { asset = 'scr_indep_fireworks', coords = vec3(110.0, 210.0, 30.0), height = 60.0, wait = 4000 },
        },
    },
}
```

---

## Comandos

| Comando | Permissão | Descrição |
|---|---|---|
| `/startshow <Location>` | `group.admin` | Inicia o show cujo nome corresponde à chave em `shows`. Nome inválido não faz nada |

---

## Entrypoints para outros recursos

### Export `StartShow` (servidor)

Dispara um show pelo nome. Mesma função usada pelo `/startshow`.

```lua
exports.qbx_fireworks:StartShow('lsia')
```

### Evento `qbx_fireworks:server:spawnObject`

Cria o prop do fogo e marca a statebag de inicialização. Usado internamente pelo cliente após a barra de progresso.

```lua
TriggerServerEvent('qbx_fireworks:server:spawnObject', 'ind_prop_firework_03', coords)
```

### Evento `qbx_fireworks:server:spawnShowObject`

Cria o prop de um fogo de show, um metro abaixo das coordenadas passadas.

```lua
TriggerEvent('qbx_fireworks:server:spawnShowObject', 'ind_prop_firework_03', coords)
```

### Evento `qbx_fiureworks:client:startShow`

Roda os efeitos de uma sequência de show no cliente. O nome do evento contém o typo `fiureworks` no código original — use exatamente assim.

```lua
TriggerClientEvent('qbx_fiureworks:client:startShow', -1, asset, coords, height, fireworkEffects)
```

---

## Localização

Strings via `ox_lib` locale, em `locales/`:

`cs`, `da`, `de`, `en`, `es`, `fr`, `ja`, `pl`, `pt`, `pt-br`, `tr`

Idioma ativo definido no `server.cfg`:

```
setr ox:locale "pt-br"
```

Chaves usadas: `placing`, `need_lighter`, `canceled`, `start_show_desc`, `start_show_param_hint`.

---

## Estrutura de arquivos

```
qbx_fireworks/
├── client/
│   └── main.lua          — callback de uso do item, efeitos de partícula, handler da statebag, evento de show
├── server/
│   └── main.lua          — registro dos itens usáveis, spawn dos props, comando /startshow, export StartShow
├── config/
│   └── shared.lua        — detonationTime, needLighter, lista de fogos e shows
├── locales/
│   └── *.json            — traduções (11 idiomas)
└── fxmanifest.lua
```
