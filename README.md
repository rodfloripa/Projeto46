


<p align="justify"><h1>Sistema de Resgate com Drones — Espiral + RL Suave</h1></p>

<p align="justify"><h2>Visão Geral</h2></p>

<p align="justify">
Este projeto simula um sistema de múltiplos drones que realizam busca e resgate em um ambiente 2D com obstáculos (árvores).
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

<p align="justify"><h2>Insight Principal</h2></p>

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

<p align="justify"><h2>Solução adotada</h2></p>

<p align="justify">
O sistema funciona bem porque segue este princípio:
</p>

<p align="justify">
<b>Use uma heurística forte para garantir comportamento global e RL apenas para ajustes locais suaves.</b>
</p>

---

<p align="justify"><h2>Estrutura do Código</h2></p>

<p align="justify"><h3>1. Configuração</h3></p>

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

<p align="justify"><h3>2. Inicialização do Ambiente</h3></p>

<p align="justify">
Responsável por criar:
</p>

<p align="justify">
• posições dos drones <br>
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

<p align="justify"><h3>3. Movimento em Espiral (Exploração)</h3></p>

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

<p align="justify"><h3>4. Campo de Repulsão (Evitar Árvores)</h3></p>

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

<p align="justify"><h3>5. Movimento do Drone</h3></p>

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

<p align="justify"><h3>6. RL (Ajuste Fino)</h3></p>

<p align="justify">
O RL atua apenas como correção de trajetória.
</p>

```python
def choose_action(self, state):
    if random.random() < EPSILON:
        return random.choice(ACTIONS)

    return max(ACTIONS, key=lambda a: self.Q.get((state,a),0))
```

```python
def update_q(self, s, a, r, s2):
    best = max([self.Q.get((s2,a2),0) for a2 in ACTIONS])

    self.Q[(s,a)] = self.Q.get((s,a),0) + \
        ALPHA * (r + GAMMA*best - self.Q.get((s,a),0))
```

<p align="justify">
Importante:
</p>

<p align="justify">
• RL não controla tudo <br>
• apenas ajusta o ângulo
</p>

---

<p align="justify"><h3>7. Máquina de Estados</h3></p>

<p align="justify">
Controla o comportamento do drone:
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

<p align="justify"><h3>8. Renderização</h3></p>

<p align="justify">
Responsável pelo GIF final.
</p>

```python
imageio.mimsave(filename, frames, fps=8)
```

---

<p align="justify"><h2>Conclusão</h2></p>

<p align="justify">
Este projeto mostra um ponto importante em sistemas reais:
</p>

<p align="justify">
<b>RL puro raramente resolve problemas complexos de navegação.</b><br>
<b>Combinar heurísticas + RL leve é muito mais eficaz.</b>
</p>

---

<p align="justify"><h2>Comparação</h2></p>


| Abordagem             | Resultado |
| --------------------- | --------- |
| RL puro               | instável  |
| Heurística pura       | funciona  |
| Heurística + RL suave | melhor    |


---

<p align="justify"><h2>Possíveis Melhorias</h2></p>

<p align="justify">
• RL treinado offline <br>
• Planejamento com A* <br>
• Mapa de cobertura global <br>
• Coordenação entre drones
</p>

---

<p align="justify"><h2>Insight Final</h2></p>

<p align="justify">
O segredo não é usar RL em tudo, mas saber onde ele realmente agrega valor.
</p>

<p align="justify">
Aqui, ele funciona melhor como:<br>
<b>um refinador, não um controlador principal.</b>
</p>
```
![Alt Text](https://github.com/rodfloripa/Projeto46/blob/main/with_rl(1).gif)
---

