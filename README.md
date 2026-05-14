
<p align="justify"><h1>Sistema de Resgate com eVTOLs — Espiral + RL Suave</h1></p>

---

<p align="justify"><h2>1. Visão Geral</h2></p>

<p align="justify">
Este projeto simula um sistema de múltiplos eVTOLs responsáveis por operações de busca e resgate em um ambiente bidimensional contendo obstáculos naturais representados por árvores. O principal objetivo é investigar como técnicas híbridas de navegação podem produzir comportamento mais estável, eficiente e robusto do que abordagens puramente baseadas em aprendizado por reforço. A arquitetura combina heurísticas geométricas clássicas com um módulo leve de <b>Reinforcement Learning (RL)</b>, permitindo criar um sistema robusto, eficiente e computacionalmente simples.
</p>

<p align="justify">
A abordagem integra três componentes principais. O primeiro é uma heurística geométrica forte baseada em exploração em espiral, responsável por garantir cobertura sistemática do espaço. O segundo é um campo contínuo de repulsão, utilizado para evitar colisões com obstáculos. O terceiro componente é um módulo leve de Q-Learning tabular, cuja função é realizar pequenos ajustes locais na trajetória dos agentes. Ao invés de utilizar RL como controlador global, o projeto explora a ideia de utilizar aprendizado por reforço apenas como refinador local da navegação.
</p>

---

<p align="justify"><h2>2. Insight Principal</h2></p>

<p align="justify">
Aplicar Reinforcement Learning puro em problemas contínuos de navegação raramente produz bons resultados sem uma enorme quantidade de engenharia adicional. Em ambientes reais, os agentes precisam lidar simultaneamente com espaços contínuos, dinâmica não-linear, obstáculos variáveis e recompensas extremamente difíceis de definir corretamente. Na prática, isso frequentemente produz trajetórias caóticas, comportamento instável e convergência extremamente lenta.
</p>

<p align="justify">
Sem uma heurística forte guiando o movimento global, o agente tende a explorar regiões irrelevantes do ambiente, repetir movimentos ineficientes e falhar na cobertura adequada do espaço. Além disso, pequenas mudanças no ambiente podem degradar significativamente o desempenho do modelo. Por esse motivo, o sistema proposto adota uma estratégia híbrida onde a inteligência global da exploração é controlada por uma heurística forte e determinística, enquanto o RL atua apenas como um mecanismo adaptativo local responsável por suavizar trajetórias e melhorar pequenas decisões de navegação.
</p>

---

<p align="justify"><h2>3. Solução Adotada</h2></p>

<p align="justify">
O princípio central deste projeto consiste em utilizar heurísticas clássicas para controlar o comportamento global do sistema e aplicar Reinforcement Learning apenas nos pontos onde ele realmente agrega valor. Em vez de aprender toda a missão do zero, os eVTOLs já possuem um comportamento exploratório eficiente definido geometricamente através de trajetórias em espiral. Essa exploração sistemática garante que o ambiente seja coberto de maneira organizada e previsível.
</p>

<p align="justify">
Sobre essa estrutura determinística, o Q-Learning aprende pequenos desvios angulares que ajudam os agentes a evitar obstáculos, suavizar curvas e reduzir o tempo necessário para alcançar objetivos específicos. Essa separação clara entre comportamento global e ajuste local reduz drasticamente o espaço de busca do aprendizado por reforço, tornando o sistema mais estável, interpretável e computacionalmente eficiente.
</p>

---

<p align="justify"><h2>4. Estrutura do Código</h2></p>

---

<p align="justify"><h3>4.1 Configuração</h3></p>

<p align="justify">
A primeira etapa do sistema define os parâmetros físicos do ambiente e os hiperparâmetros associados ao módulo de aprendizado por reforço. Esses valores controlam características importantes da simulação, como tamanho do ambiente, quantidade de agentes, resolução do movimento, alcance dos sensores e política de exploração utilizada pelo RL.
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

<p align="justify">
Os parâmetros escolhidos permitem criar um ambiente suficientemente complexo para validar o comportamento híbrido entre heurísticas geométricas e aprendizado adaptativo.
</p>

---

<p align="justify"><h3>4.2 Inicialização do Ambiente</h3></p>

<p align="justify">
A função de inicialização é responsável por criar o estado inicial completo da simulação. Todos os eVTOLs começam posicionados na base central, enquanto árvores e vítimas são distribuídas ao longo do ambiente. Além das posições iniciais, cada agente recebe parâmetros associados ao movimento em espiral, como ângulo inicial, raio de exploração e direção radial.
</p>

```python
def reset(self):
    self.pos = {i: BASE.copy() for i in range(N_DRONES)}

    self.theta = {i: i * (2*np.pi / N_DRONES) for i in range(N_DRONES)}
    self.radius = {i: 1.0 for i in range(N_DRONES)}

    self.trees = [...]
    self.victims = [...]
```

<p align="justify">
Esses parâmetros garantem que os agentes iniciem a exploração de maneira coordenada, evitando sobreposição excessiva entre trajetórias.
</p>

---

<p align="justify"><h3>4.3 Movimento em Espiral (Exploração)</h3></p>

<p align="justify">
O movimento em espiral representa o núcleo estratégico do sistema. Ele garante cobertura sistemática do ambiente sem depender exclusivamente de aprendizado. A trajetória é construída ajustando continuamente o raio e o ângulo de cada agente em relação à base central.
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
Sem essa heurística forte, o RL teria enorme dificuldade para explorar eficientemente o ambiente. A espiral atua como um prior geométrico extremamente poderoso, fornecendo comportamento global organizado e previsível.
</p>

---

<p align="justify"><h3>4.4 Campo de Repulsão (Evitar Árvores)</h3></p>

<p align="justify">
O sistema de repulsão utiliza uma analogia inspirada em campos físicos para evitar colisões. Quando um eVTOL se aproxima de uma árvore, uma força vetorial é aplicada na direção oposta ao obstáculo. Quanto menor a distância, maior a intensidade da repulsão.
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
Essa abordagem produz desvios suaves e contínuos, evitando mudanças abruptas de direção e gerando comportamento significativamente mais natural durante a navegação.
</p>

---

<p align="justify"><h3>4.5 Movimento do eVTOL</h3></p>

<p align="justify">
O movimento final do agente é resultado da combinação entre direção desejada, repulsão de obstáculos e correção angular produzida pelo RL. Inicialmente o agente calcula o vetor principal até o alvo ou waypoint. Em seguida, o módulo de aprendizado aplica pequenos ajustes na direção calculada.
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

<p align="justify">
A combinação desses componentes produz trajetórias mais suaves, estáveis e eficientes do que abordagens puramente reativas ou totalmente baseadas em RL.
</p>

---

<p align="justify"><h3>4.6 RL (Ajuste Fino)</h3></p>

<p align="justify">
Neste projeto, o Reinforcement Learning não é utilizado como controlador global da navegação. Seu papel é funcionar apenas como um refinador local de trajetória. O sistema principal continua sendo controlado pela heurística geométrica em espiral, enquanto o RL aprende pequenos desvios angulares capazes de melhorar a movimentação em regiões complexas do ambiente.
</p>

<p align="justify">
Ao limitar o escopo do aprendizado apenas para ajustes locais, o problema se torna muito mais simples e estável. Em vez de aprender toda a política de exploração, o agente aprende apenas como corrigir levemente sua direção para reduzir colisões e melhorar eficiência.
</p>

---

<p align="justify"><h4>4.6.1 Estrutura da Tabela Q</h4></p>

<p align="justify">
Cada eVTOL possui sua própria tabela Q. Essa estrutura armazena o valor esperado de recompensa associado a pares de estado e ação. A implementação utiliza um dicionário Python simples, permitindo acesso rápido e atualização eficiente dos valores aprendidos.
</p>

```python
self.Q = {}
```

<p align="justify">
A chave do dicionário representa o estado atual do agente juntamente com a ação escolhida. O valor associado corresponde à qualidade esperada daquela decisão específica.
</p>

---

<p align="justify"><h4>4.6.2 Representação do Estado</h4></p>

<p align="justify">
Como o ambiente é contínuo, a posição do agente precisa ser discretizada para permitir o uso de Q-Learning tabular. Isso é realizado dividindo o espaço em células de tamanho fixo.
</p>

```python
def get_state(self, i):
    return tuple(np.round(self.pos[i] / 5).astype(int))
```

<p align="justify">
Sem essa discretização, o número de estados possíveis seria praticamente infinito, tornando inviável o armazenamento da tabela Q. A discretização reduz drasticamente a complexidade do problema mantendo informação espacial suficiente para aprendizado local eficiente.
</p>

---

<p align="justify"><h4>4.6.3 Espaço de Ações</h4></p>

<p align="justify">
As ações do agente não representam movimentos absolutos. Elas representam pequenos ajustes angulares aplicados sobre a direção desejada calculada geometricamente.
</p>

```python
ACTIONS = [-0.5, -0.2, 0, 0.2, 0.5]
```

<p align="justify">
Valores positivos produzem rotações em uma direção, enquanto valores negativos produzem rotações opostas. A ação zero mantém o movimento original sem alteração. Esse conjunto reduzido de ações simplifica enormemente o processo de aprendizado, permitindo que o agente aprenda correções locais de trajetória de forma eficiente.
</p>

---

<p align="justify"><h4>4.6.4 Estratégia Epsilon-Greedy</h4></p>

<p align="justify">
A política de decisão utiliza a estratégia epsilon-greedy. Durante a navegação, o agente alterna entre exploração e explotação. Em alguns momentos escolhe ações aleatórias para descobrir novas possibilidades; em outros utiliza a melhor ação conhecida para o estado atual.
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
Esse equilíbrio é essencial para evitar convergência prematura e permitir aprendizado contínuo ao longo da simulação.
</p>

---

<p align="justify"><h4>4.6.5 Atualização da Tabela Q</h4></p>

<p align="justify">
Após executar uma ação, o agente calcula uma recompensa baseada na distância até o alvo e atualiza sua tabela utilizando a equação clássica do Q-Learning.
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

<p align="justify">
A recompensa é definida utilizando a distância negativa até o alvo. Quanto menor a distância, maior a recompensa recebida pelo agente.
</p>

```python
r = -np.linalg.norm(self.pos[i] - target)
```

---

<p align="justify"><h4>4.6.6 Integração do RL com a Navegação</h4></p>

<p align="justify">
O RL é integrado diretamente ao vetor de direção calculado geometricamente. O agente primeiro determina a direção ideal até o alvo e posteriormente aplica um pequeno ajuste angular aprendido.
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
Após o movimento, o estado seguinte é observado e a tabela Q é atualizada continuamente.
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
O aprendizado ocorre online durante toda a simulação.
</p>

---

<p align="justify"><h4>4.6.7 Papel do RL no Sistema</h4></p>

<p align="justify">
Sem o módulo de RL, os eVTOLs apenas seguiriam o waypoint geométrico definido pela espiral. Com Q-Learning, os agentes aprendem pequenos desvios locais capazes de melhorar significativamente a navegação em regiões complexas. O aprendizado reduz colisões, suaviza trajetórias e melhora eficiência de movimentação sem comprometer a estabilidade global fornecida pela heurística principal.
</p>

---

<p align="justify"><h3>4.7 Máquina de Estados</h3></p>

<p align="justify">
A máquina de estados controla o comportamento global de cada agente durante a missão. Dependendo da situação atual, o eVTOL alterna entre exploração, deslocamento até vítimas e retorno à base.
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
Essa organização modular simplifica a lógica do sistema e torna o comportamento dos agentes mais previsível e interpretável.
</p>

---

<p align="justify"><h3>4.8 Renderização</h3></p>

<p align="justify">
A etapa final é responsável pela geração do GIF da simulação. Cada frame representa o estado atual do ambiente e das trajetórias dos eVTOLs.
</p>

```python
imageio.mimsave(filename, frames, fps=8)
```

<p align="justify">
Isso permite visualizar claramente o comportamento emergente produzido pela integração entre heurísticas geométricas e aprendizado por reforço.
</p>

---

<p align="justify"><h2>5. Conclusão</h2></p>

<p align="justify">
Este projeto demonstra um princípio extremamente importante em sistemas inteligentes reais: Reinforcement Learning raramente funciona melhor quando utilizado isoladamente em problemas contínuos complexos. A combinação entre heurísticas fortes e aprendizado leve produz sistemas significativamente mais estáveis, eficientes e interpretáveis.
</p>

<p align="justify">
Ao utilizar RL apenas como refinador local, o sistema reduz complexidade computacional e melhora robustez da navegação. Isso permite criar agentes capazes de operar em ambientes complexos mantendo comportamento previsível e eficiente.
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
O sistema ainda pode ser expandido através de treinamento offline mais sofisticado, planejamento global utilizando algoritmos como A*, mapas globais de cobertura e coordenação cooperativa entre agentes. Essas melhorias permitiriam aumentar escalabilidade e eficiência em ambientes maiores e mais complexos.
</p>

---

<p align="justify"><h2>8. Insight Final</h2></p>

<p align="justify">
O principal insight deste projeto é que o verdadeiro poder do Reinforcement Learning aparece quando ele é aplicado estrategicamente nos pontos corretos do sistema. Aqui, o RL funciona melhor não como controlador global da navegação, mas como um refinador inteligente capaz de melhorar pequenas decisões locais sem comprometer a estabilidade estrutural fornecida pelas heurísticas geométricas.
</p>

![Alt Text](https://github.com/rodfloripa/Projeto46/blob/main/rl.gif)

```
```
