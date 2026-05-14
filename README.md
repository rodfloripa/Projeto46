
<p align="justify"><h1>Sistema de Resgate com eVTOLs — Espiral + RL Suave</h1></p>

---

<p align="justify"><h2>1. Visão Geral</h2></p>

<p align="justify">
Este projeto simula um sistema de múltiplos eVTOLs que realizam busca e resgate em um ambiente 2D com obstáculos (árvores).
</p>

<p align="justify">
A abordagem combina:
</p>

<p align="justify">
• Heurística forte (espiral) → garante cobertura do espaço <br>
• Campo de repulsão → evita colisões <br>
• Reinforcement Learning (RL) leve → faz ajustes finos na trajetória
</p>

---

<p align="justify"><h2>2. Insight Principal</h2></p>

<p align="justify">
RL puro não funciona bem nesse problema.
</p>

<p align="justify">
Motivos:
</p>

<p align="justify">
• Espaço contínuo (movimento em 2D) <br>
• Recompensa difícil de definir <br>
• Exploração ineficiente <br>
• Convergência lenta ou inexistente
</p>

<p align="justify">
Resultado prático:
</p>

<p align="justify">
• RL puro gera trajetórias caóticas <br>
• Não cobre o espaço de forma eficiente <br>
• Pode ficar preso ou nunca encontrar vítimas
</p>

---

<p align="justify"><h2>3. Solução adotada</h2></p>

<p align="justify">
O sistema funciona bem porque segue este princípio:
</p>

<p align="justify">
<b>Use uma heurística forte para garantir comportamento global e RL apenas para ajustes locais suaves.</b>
</p>

---

<p align="justify"><h2>4. Estrutura do Código</h2></p>

---

<p align="justify"><h3>4.1 Configuração</h3></p>

<p align="justify">
Define parâmetros do ambiente e do RL.
</p>

```python
GRID = 60
N_DRONES = 3
N_VICTIMS = 6
N_TREES = 6

STEP = 0.7
SENSOR_RANGE = 1.5

ACTIONS = [-0.5, -0.2, 0, 0.2, 0.5]
EPSILON = 0.1
````

---

<p align="justify"><h3>4.2 Inicialização do Ambiente</h3></p>

<p align="justify">
Responsável por criar:
</p>

<p align="justify">
• posições dos eVTOLs <br>
• árvores <br>
• vítimas
</p>

```python
def reset(self):
    self.pos = {i: BASE.copy() for i in range(N_DRONES)}

    self.theta = {i: i * (2*np.pi / N_DRONES) for i in range(N_DRONES)}
    self.radius = {i: 1.0 for i in range(N_DRONES)}

    self.trees = [...]
    self.victims = [...]
```

---

<p align="justify"><h3>4.3 Movimento em Espiral (Exploração)</h3></p>

<p align="justify">
A espiral garante cobertura sistemática do ambiente.
</p>

```python
def update_spiral(self, i):

    if self.radius[i] >= R_MAX:
        self.radial_dir[i] = -1
    elif self.radius[i] <= R_MIN:
        self.radial_dir[i] = 1

    self.radius[i] += self.radial_dir[i] * R_STEP
    self.theta[i] += 0.5

    self.waypoint[i] = np.array([
        BASE[0] + self.radius[i] * np.cos(self.theta[i]),
        BASE[1] + self.radius[i] * np.sin(self.theta[i])
    ])
```

<p align="justify">
Esse é o coração do sistema. Sem isso, o RL não consegue explorar de forma eficiente.
</p>

---

<p align="justify"><h3>4.4 Campo de Repulsão (Evitar Árvores)</h3></p>

<p align="justify">
Evita colisões de forma contínua.
</p>

```python
def repulsion(self, pos):
    force = np.zeros(2)

    for t in self.trees:
        diff = pos - t
        dist = np.linalg.norm(diff)

        if dist < REPULSION_RADIUS:
            force += (diff/dist) * (REPULSION_STRENGTH / dist)

    return force
```

<p align="justify">
Funciona como um campo físico.
</p>

---

<p align="justify"><h3>4.5 Movimento do eVTOL</h3></p>

<p align="justify">
Combina:
</p>

<p align="justify">
• direção desejada (espiral ou alvo) <br>
• desvio de obstáculos <br>
• ajuste do RL
</p>

```python
def move(self, i, target):

    desired = target - self.pos[i]
    desired /= np.linalg.norm(desired)

    if self.use_rl:
        state = self.get_state(i)
        action = self.choose_action(state)

        angle = np.arctan2(desired[1], desired[0])
        angle += action

        desired = np.array([np.cos(angle), np.sin(angle)])

    direction = desired + self.repulsion(self.pos[i])
    direction /= np.linalg.norm(direction)

    self.pos[i] += direction * STEP
```

---

<p align="justify"><h3>4.6 RL (Ajuste Fino)</h3></p>

<p align="justify">

Neste projeto, o módulo de <b>Reinforcement Learning (RL)</b> atua apenas como um sistema de correção local da trajetória.

O objetivo não é fazer o eVTOL aprender toda a missão do zero, mas sim aprender pequenos desvios angulares que melhoram a navegação em regiões com obstáculos.

A lógica principal da navegação continua sendo controlada pela heurística em espiral. O RL funciona como uma camada adaptativa responsável por pequenos refinamentos na direção do movimento.

</p>

---

<p align="justify"><h4>4.6.1 Estrutura da Tabela Q</h4></p>

<p align="justify">

Cada eVTOL possui sua própria tabela Q, responsável por armazenar os valores esperados de recompensa para pares <b>(estado, ação)</b>.

A estrutura é implementada como um dicionário Python:

</p>

```python
self.Q = {}
```

<p align="justify">

A chave do dicionário é composta por:

• estado atual do eVTOL <br>
• ação escolhida naquele estado

O valor armazenado representa a qualidade esperada daquela ação naquele contexto específico.

</p>

---

<p align="justify"><h4>4.6.2 Representação do Estado</h4></p>

<p align="justify">

O ambiente físico é contínuo, mas Q-Learning tabular funciona melhor em espaços discretos. Para resolver isso, a posição do eVTOL é convertida para um grid reduzido.

</p>

```python
def get_state(self, i):
    return tuple(np.round(self.pos[i] / 5).astype(int))
```

<p align="justify">

A posição contínua é dividida por 5 e arredondada, transformando o espaço em células discretas.

Sem essa discretização, o número de estados seria praticamente infinito, inviabilizando o uso de Q-Learning tabular.

Essa estratégia reduz drasticamente a complexidade do problema e permite aprendizado eficiente mesmo com poucos episódios.

</p>

---

<p align="justify"><h4>4.6.3 Espaço de Ações</h4></p>

<p align="justify">

As ações não representam movimentos absolutos. Em vez disso, representam pequenos ajustes angulares aplicados ao vetor de direção desejado.

</p>

```python
ACTIONS = [-0.5, -0.2, 0, 0.2, 0.5]
```

| Ação   | Significado          |
| ------ | -------------------- |
| `0`    | Segue reto           |
| `0.2`  | Pequena curva        |
| `0.5`  | Curva mais agressiva |
| `-0.2` | Ajuste oposto        |
| `-0.5` | Curva forte oposta   |

<p align="justify">

O RL não decide o destino do eVTOL. Ele apenas aprende como ajustar a direção localmente para evitar colisões e reduzir o caminho até o alvo.

</p>

---

<p align="justify"><h4>4.6.4 Estratégia Epsilon-Greedy</h4></p>

<p align="justify">

A escolha das ações utiliza a estratégia <b>epsilon-greedy</b>, muito comum em aprendizado por reforço.

</p>

```python
def choose_action(self, state):

    if random.random() < EPSILON:
        return random.choice(ACTIONS)

    return max(
        ACTIONS,
        key=lambda a: self.Q.get((state, a), 0)
    )
```

<p align="justify">

O comportamento do agente é dividido em duas partes:

• exploração → testa ações aleatórias <br>
• explotação → utiliza a melhor ação conhecida

Com <b>EPSILON = 0.1</b>:

• 10% das vezes o eVTOL explora <br>
• 90% das vezes utiliza o conhecimento aprendido

Esse equilíbrio é essencial para evitar que o agente fique preso em soluções ruins.

</p>

---

<p align="justify"><h4>4.6.5 Atualização da Tabela Q</h4></p>

<p align="justify">

Após executar uma ação e observar o novo estado, o eVTOL atualiza sua tabela Q usando a equação clássica do Q-Learning.

</p>

```python
def update_q(self, s, a, r, s2):

    best = max([
        self.Q.get((s2, a2), 0)
        for a2 in ACTIONS
    ])

    self.Q[(s, a)] = (
        self.Q.get((s, a), 0)
        + ALPHA * (
            r + GAMMA * best
            - self.Q.get((s, a), 0)
        )
    )
```

```python
Q(s,a) = Q(s,a) + α * [r + γ * max(Q(s',a')) - Q(s,a)]
```

| Parâmetro   | Função                       |
| ----------- | ---------------------------- |
| `α (ALPHA)` | Taxa de aprendizado          |
| `γ (GAMMA)` | Peso das recompensas futuras |
| `r`         | Recompensa imediata          |
| `s'`        | Próximo estado               |

```python
ALPHA = 0.1
GAMMA = 0.9
```

```python
r = -np.linalg.norm(self.pos[i] - target)
```

<p align="justify">

Quanto mais próximo do alvo, maior a recompensa (menos negativa).

</p>

---

<p align="justify"><h4>4.6.6 Integração do RL com a Navegação</h4></p>

<p align="justify">

O RL é integrado diretamente ao sistema de movimento do eVTOL.

Inicialmente o agente calcula a direção geométrica ideal até o alvo. Em seguida, o RL aplica um pequeno ajuste angular aprendido.

</p>

```python
if self.use_rl:

    state = self.get_state(i)
    action = self.choose_action(state)

    angle = np.arctan2(
        desired[1],
        desired[0]
    )

    angle += action

    desired = np.array([
        np.cos(angle),
        np.sin(angle)
    ])
```

<p align="justify">

Após o movimento:

</p>

```python
if self.use_rl:

    r = -np.linalg.norm(
        self.pos[i] - target
    )

    s2 = self.get_state(i)

    self.update_q(
        state,
        action,
        r,
        s2
    )
```

<p align="justify">

O aprendizado acontece continuamente durante a simulação.

O eVTOL executa ações, observa os resultados e ajusta gradualmente sua política de navegação.

</p>

---

<p align="justify"><h4>4.6.7 Papel do RL no Sistema</h4></p>

<p align="justify">

Sem o módulo de RL, os eVTOLs simplesmente seguiriam a direção do waypoint utilizando apenas heurísticas clássicas de navegação.

Com Q-Learning:

• os eVTOLs aprendem desvios locais mais eficientes <br>
• melhoram o contorno de obstáculos <br>
• reduzem colisões <br>
• encontram trajetórias mais suaves <br>
• diminuem o tempo de chegada ao alvo

O RL atua como uma camada adaptativa sobre a lógica principal do sistema.

Isso torna o comportamento significativamente mais robusto em ambientes complexos, mantendo ao mesmo tempo uma arquitetura leve e computacionalmente simples.

</p>

---

<p align="justify">

Importante:

</p>

<p align="justify">

• RL não controla tudo <br>
• apenas ajusta o ângulo

</p>

---

<p align="justify"><h3>4.7 Máquina de Estados</h3></p>

<p align="justify">
Controla o comportamento do eVTOL:
</p>

```python
if self.state[i] == "search":
    ...
elif self.state[i] == "to_victim":
    ...
elif self.state[i] == "to_base":
    ...
```

<p align="justify">
Estados:
</p>

<p align="justify">
• search → exploração em espiral <br>
• to_victim → ir até a vítima <br>
• to_base → retornar
</p>

---

<p align="justify"><h3>4.8 Renderização</h3></p>

<p align="justify">
Responsável pelo GIF final.
</p>

```python
imageio.mimsave(filename, frames, fps=8)
```

---

<p align="justify"><h2>5. Conclusão</h2></p>

<p align="justify">
Este projeto mostra um ponto importante em sistemas reais:
</p>

<p align="justify">
<b>RL puro raramente resolve problemas complexos de navegação.</b><br>
<b>Combinar heurísticas + RL leve é muito mais eficaz.</b>
</p>

---

<p align="justify"><h2>6. Comparação</h2></p>

| Abordagem             | Resultado |
| --------------------- | --------- |
| RL puro               | instável  |
| Heurística pura       | funciona  |
| Heurística + RL suave | melhor    |

---

<p align="justify"><h2>7. Possíveis Melhorias</h2></p>

<p align="justify">
• RL treinado offline <br>
• Planejamento com A* <br>
• Mapa de cobertura global <br>
• Coordenação entre eVTOLs
</p>

---

<p align="justify"><h2>8. Insight Final</h2></p>

<p align="justify">
O segredo não é usar RL em tudo, mas saber onde ele realmente agrega valor.
</p>

<p align="justify">
Aqui, ele funciona melhor como:<br>
<b>um refinador, não um controlador principal.</b>
</p>

![Alt Text](https://github.com/rodfloripa/Projeto46/blob/main/rl.gif)

```
```



