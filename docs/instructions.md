# Fly-in — Referênciaa

Resumo de tudo que o subject exige, na ordem em que você vai precisar.

## 1. Objetivo
Rotear uma frota de drones de uma zona **start** até uma zona **end** no **menor número de turnos** possível, respeitando capacidades e custos de movimento. Entrada: um grafo de zonas + restrições.

## 2. Regras gerais (bloqueantes na avaliação)
- Python **3.10+**
- **flake8** limpo e **mypy** limpo (todas as funções)
- Projeto **100% orientado a objetos** e **100% type-safe**
- **Proibido** qualquer lib que ajude com grafos (networkx, graphlib, etc.)
- Type hints em parâmetros, retornos e variáveis
- Docstrings PEP 257 (Google ou NumPy)
- Tratar exceções (`try/except`); recursos com context managers
- **Crash por exceção não tratada na review = projeto não funcional**
- Testes (pytest/unittest) são recomendados, mas **não são entregues nem avaliados**
- `.gitignore` para artefatos Python; venv recomendado
- Todos os arquivos na **raiz** do repositório

## 3. Makefile (regras obrigatórias)
| Regra | O que faz |
|---|---|
| `install` | instala dependências (pip, uv, pipx...) |
| `run` | roda o script principal |
| `debug` | roda no debugger (pdb) |
| `clean` | remove `__pycache__`, `.mypy_cache` etc. |
| `lint` | `flake8 .` + `mypy . --warn-return-any --warn-unused-ignores --ignore-missing-imports --disallow-untyped-defs --check-untyped-defs` |
| `lint-strict` (opcional) | `flake8 .` + `mypy . --strict` |

## 4. Formato do arquivo de entrada
```
nb_drones: 5

start_hub: hub 0 0 [color=green]
end_hub: goal 10 10 [color=yellow]
hub: roof1 3 4 [zone=restricted color=red]
hub: roof2 6 2 [zone=normal color=blue]
hub: corridorA 4 3 [zone=priority color=green max_drones=2]
hub: tunnelB 7 4 [zone=normal color=red]
hub: obstacleX 5 5 [zone=blocked color=gray]
connection: hub-roof1
connection: hub-corridorA
connection: roof1-roof2
connection: roof2-goal
connection: corridorA-tunnelB [max_link_capacity=2]
connection: tunnelB-goal
```
- 1ª linha: `nb_drones: <número>`
- Zonas: `start_hub:` / `end_hub:` / `hub:` + `<name> <x> <y> [metadata]`
- Metadata opcional, entre `[...]`, tags em **qualquer ordem**:
  - `zone=<type>` (default `normal`)
  - `color=<valor>` (default nenhum; qualquer palavra única, sem lista fixa)
  - `max_drones=<n>` (default 1)
- Conexões: `connection: <a>-<b> [max_link_capacity=<n>]` (default 1), **bidirecional**
- Comentários começam com `#` e são ignorados
- Coordenadas sempre inteiras; sempre 1 start e 1 end

## 5. Tipos de zona (custo = tipo do **destino**)
| Tipo | Custo | Obs |
|---|---|---|
| `normal` | 1 turno | default |
| `priority` | 1 turno | deve ser **preferida** no pathfinding |
| `restricted` | 2 turnos | |
| `blocked` | — | não pode entrar nem passar; caminho com ela é inválido |

## 6. Ocupação
- Zona: no máximo **1 drone** por turno (ou `max_drones=N`)
- Exceções: **start** (todos começam lá) e **end** (qualquer quantidade chega e é "entregue")
- `max_drones` é **ignorado** em start/end (não é erro de validação)
- Dois drones não entram na mesma zona no mesmo turno se estourar a capacidade
- `max_link_capacity` limita quantos drones usam a mesma conexão ao mesmo tempo

## 7. Movimento e turnos
A cada turno, cada drone pode: **mover** para zona adjacente, **entrar numa conexão** rumo a zona restricted, ou **ficar parado**.

- Movimentos são **simultâneos**: todos decididos a partir do **estado no início do turno** (sem mudar de ideia no meio)
- Vaga liberada neste turno **já pode ser usada** por outro drone no mesmo turno (fila indiana avança 1 passo por turno)
- Zona precisa ter capacidade depois que todos os que saem liberarem espaço
- **Restricted**: o drone ocupa a conexão durante o trânsito e **DEVE chegar** no turno seguinte. **Não pode esperar na conexão**
- A conexão restricted é liberada no turno em que o drone chega; outro drone pode começar a cruzar **nesse mesmo turno** (se o resto permitir)

Exemplo (link_capacity=1 para zona restricted):
```
D1-restrictedConn
D1-restrictedZone D2-restrictedConn
D2-restrictedZone
```

## 8. Parser — o que validar
- 1ª linha `nb_drones: <inteiro positivo>`; qualquer quantidade de drones deve funcionar
- Exatamente 1 `start_hub` e 1 `end_hub`
- Nome único, coordenadas inteiras; nome **sem traço e sem espaço**
- Conexão só entre zonas **já definidas** (antes da linha da conexão)
- Conexão duplicada é erro (`a-b` == `b-a`)
- Metadata sintaticamente válida
- Tipo de zona inválido = erro de parsing
- `max_drones` / `max_link_capacity` = inteiros positivos
- Qualquer outro erro: parar e mostrar mensagem clara com **linha e causa**
- Mapa válido mas **sem caminho start→end**: não é erro de parsing, mas o programa deve **detectar e reportar** (sem crash, sem loop infinito)
- Recomendado criar seus próprios mapas de teste (casos extremos e inválidos)

## 9. Formato de saída
- **1 linha por turno**, movimentos separados por espaço
- `D<ID>-<zone>` (destino) ou `D<ID>-<connection>` (drone em trânsito para restricted)
- Drones que **não se movem** são omitidos da linha
- Drones que chegam ao end saem do rastreamento
- Termina quando todos chegaram

```
D1-roof1 D2-corridorA
D1-roof2 D2-tunnelB
D1-goal D2-goal
```
> A grafia exata do nome da `<connection>` não está clara no subject (o exemplo usa `restrictedConn`). Vale confirmar com um peer/staff.

## 10. Algoritmo — o que precisa tratar
- Distribuir drones por **múltiplos caminhos**
- **Esperar** estrategicamente
- Evitar **conflitos e deadlocks**
- Considerar: custos por tipo de zona, agendamento por turno, estrutura do grafo (caminhos disjuntos/sobrepostos), `max_drones` e `max_link_capacity`
- Ser **adaptável**: mapas diferentes podem pedir estratégias diferentes
- Perguntas que o subject sugere pensar: eficiência, muitos drones, complexidade (O(...)), cache/recálculo de caminhos, uso de memória, como o visual ajuda

## 11. Representação visual (obrigatória)
Terminal colorido (movimentos e estados das zonas) **e/ou** interface gráfica. Quando `color=` for dado, deve aparecer no visual.

## 12. Benchmarks (só nos mapas do subject)
| Nível | Mapa | Drones | Ótimo |
|---|---|---|---|
| Easy | Linear path | 2 | 4 |
| Easy | Simple fork | 4 | 4 |
| Easy | Basic capacity | 4 | 4 |
| Medium | Dead end trap | 5 | 8 |
| Medium | Circular loop | 6 | 10 |
| Medium | Priority puzzle | 5 | 6 |
| Hard | Maze nightmare | 8 | 13 |
| Hard | Capacity hell | 12 | 16 |
| Hard | Ultimate challenge | 15 | 26 |
| Challenger (opcional) | The Impossible Dream | 25 | 43 |

- Correção é avaliada **separadamente** da velocidade; velocidade **nunca reprova**
- Pontos: todos os mapas rodando corretamente → pontos por ficar **dentro de 25% do ótimo em todos** → pontos extras por atingir o **ótimo exato em todos**
- Impossible Dream: limitado por gates de capacidade 1; ótimo = latência de entrada (19) + 1 turno por drone restante (24) = **43**
- Mapas da avaliação podem ser **diferentes** dos do subject

## 13. README.md (obrigatório, em inglês, na raiz)
1. **1ª linha em itálico**: `*This project has been created as part of the 42 curriculum by mpinto-l.*`
2. Seção **Description** (objetivo + visão geral)
3. Seção **Instructions** (instalação/execução)
4. Seção **Resources** (referências clássicas + **como a IA foi usada**, em quais tarefas e partes)
5. Descrição detalhada das **escolhas de algoritmo** e estratégia de implementação
6. Documentação das **features visuais** e como melhoram a experiência
7. **Exemplo de entrada e saída esperada**

## 14. Bônus
Só é avaliado se o mandatório estiver 100%: resolver o **Impossible Dream em 43 turnos**.

## 15. Entrega e peer-review
Entregar: parser, engine de simulação, pathfinding, sistema visual e saída no formato especificado. Na avaliação podem pedir para **explicar o código** ou **fazer uma pequena modificação** (função, display, estrutura de dados) em poucos minutos.

## 16. Pendência
O capítulo **II (AI Instructions, págs. 3–4)** não veio no texto que eu recebi. Leia direto no PDF, porque pode afetar o que você escreve em "Resources" sobre uso de IA.
