# Changelog — Sessões de trabalho via MCP (Unreal Editor)

Registro de todas as mudanças feitas no projeto `WheelyWiener_V0.1` através do agente de IA conectado ao MCP (`http://127.0.0.1:8000/mcp`). Cobre desde a investigação inicial de bugs de movimento até a implementação completa do mecanismo de Arremesso/Captura (Throw & Catch) do `BP_SauceMaster`, o Grab com `BP_GrabPistol` (distância pela roda do mouse, linha de debug e altura ajustável) e o Grind/Slide em rails `BP_SplineGrindMaster`.

---

## 1. Física de pulo/queda (`BPC_WheelchairMovement`)

**Problema:** o personagem flutuava ao pular — a queda era lenta demais e "sem peso".

**Correção:**
- Ajustado o valor de `ExtraFallGravity` (instância) para 200, aplicado no evento `Movement`: enquanto `bIsJump == true`, subtrai `ExtraFallGravity * DeltaTime` da velocidade Z, reforçando a gravidade durante a queda.

**Problema relacionado:** com um item/arma equipado, a queda continuava lenta mesmo após a correção acima.

**Causa raiz:** a esfera de colisão (`Sphere`) do item equipado não tinha a colisão desabilitada, atrapalhando o trace de detecção de chão do personagem.

**Correção:**
- `BPC_InteractionDetector::AttachItem`: adicionado `SetCollisionEnabled(NoCollision)` na `Sphere` do item ao equipá-lo.
- `BPC_InteractionDetector::DetachItem`: adicionado `SetCollisionEnabled(QueryOnly)` na `Sphere` do item ao soltá-lo, restaurando o valor original.

---

## 2. Rotação excessiva no ar durante o pulo (`BPC_WheelchairMovement`)

**Problema:** ao pular e virar para os lados, o personagem girava rápido demais enquanto estava no ar.

**Causa raiz:** o evento `Movement` resetava tanto `LinearDamping` quanto `AngularDamping` durante o pulo, removendo o amortecimento angular e deixando o giro sem controle.

**Correção:**
- Removida a chamada que resetava `AngularDamping` dentro do bloco `if bIsJump`; apenas `LinearDamping` continua sendo restaurado.

---

## 3. Erro do GAS ao pegar itens sem `ShotAbility` (`BPC_InteractionDetector`)

**Problema:** um `ensure`/erro do Gameplay Ability System ("invalid Ability Class") ao pegar itens que não tinham uma `ShotAbility` definida (ex.: `DA_SimpleSauce` antes de ter uma ability associada).

**Correção:**
- `AttachItem`: a chamada `GiveAbility` agora só ocorre dentro de `if (IsValidClass(ShotAbility))`.

---

## 4. Mecânica de Arremesso e Captura do Sauce (Throw & Catch)

Sistema completo implementado via GAS para permitir mirar (RMB), arremessar (LMB) e capturar o `BP_SauceMaster`, com preview de trajetória em tempo real e "quebra" (Splat) ao errar o alvo.

### 4.1 Novas Abilities

- **`GA_Throw`** (nova, `Content/Main/Core/Abilities/`)
  - `InstancingPolicy = InstancedPerExecution`, `NetExecutionPolicy = LocalPredicted`.
  - Ao ativar: calcula a trajetória com `SuggestProjectileVelocityCustomArc` (arco baixo, `ArcParam = 0.35`, `OverrideGravityZ = -1500`), desanexa o item (`DetachItem`), ignora colisão do item contra o próprio lançador (`IgnoreActorWhenMoving`), espera um tick (`DelayUntilNextTick` — necessário porque `SetPhysicsLinearVelocity` não "pega" no mesmo frame em que `SetSimulatePhysics(true)` é chamado), aplica a velocidade calculada, marca `bIsThrown = true` e `ThrowerReference`, desenha o trajeto de debug e finaliza a ability.
- **`GA_Catch`** (nova, `Content/Main/Core/Abilities/`)
  - `InstancingPolicy = InstancedPerActor`, `NetExecutionPolicy = ServerInitiated`, disparada por Gameplay Event `Event.Sauce.Catch`.
  - Ao receber o evento: destaca o item atualmente equipado (se houver) e anexa o Sauce capturado ao personagem.
  - Concedida a todo personagem no `EventBeginPlay` de `BP_CharacterMaster` (gated por autoridade do servidor).

### 4.2 `BP_SauceMaster` — novas variáveis e lógica

- Variáveis novas: `bIsThrown` (bool), `ThrowerReference` (ref. a `BP_CharacterMaster`).
- Componente novo: `CatchDetectionSphere` (raio 200, perfil `OverlapOnlyPawn`).
- `OnComponentBeginOverlap(CatchDetectionSphere)`: se `bIsThrown` e o ator sobreposto não é quem arremessou, envia `Event.Sauce.Catch` para esse ator (ativando `GA_Catch`); caso contrário (sobrepôs o próprio lançador), apenas zera `bIsThrown`.
- `EventTick`: enquanto `bIsThrown`, verifica a magnitude da velocidade física do `StaticMesh`; se cair abaixo de `5.0` (item "pousou"/parou), faz `spawn` de `BP_SplatSauce` no local e destrói o Sauce — forçando o jogador a achar outro no mapa.

### 4.3 Data Asset

- `DA_SimpleSauce`: campo `ShotAbility` agora aponta para `GA_Throw`.

### 4.4 Bug de ativação (LMB não fazia nada com o Sauce equipado)

**Causa raiz:** dentro do subgrafo colapsado "Inputs" do `EventGraph` de `BP_CharacterMaster`, o gatilho de LMB (`IA_Shoot`) só chamava a `ShotAbility` quando `AnimationChooser == Carryng` (estado de "segurando arma"). Itens arremessáveis (Sauce) usam `AnimationChooser == Throw`, então essa condição nunca era satisfeita e a ability nunca era ativada.

**Correção:**
- Conectados todos os casos do `SwitchOnE_ItemType` ao mesmo `Branch` de ativação.
- Removida a condição `AnimationChooser == Carryng` (substituída por `true`), já que ela impedia estruturalmente a ativação para itens do tipo "Throw".

---

## 5. Preview contínuo da trajetória (mirar com RMB)

Implementado no `EventTick` de `BP_CharacterMaster`: enquanto o jogador mira (RMB) com o Sauce equipado, a trajetória prevista é desenhada em tempo real (`PredictProjectilePathByTraceChannel`), atualizando a cada frame conforme a câmera se move.

### 5.1 Erros de "Accessed None" / debug não desenhava

**Causa raiz:** `BPC_InteractionDetector`/`HeldItem` podiam estar temporariamente inválidos no momento em que o Tick tentava ler a trajetória, causando erro e interrompendo o cálculo antes do desenho.

**Correção:** adicionados guardas `IsValid` (macro `Utilities|IsValid`) em cadeia — primeiro valida `BPC_InteractionDetector`, depois `HeldItem` — antes de prosseguir com o cálculo. Se inválido, o preview é simplesmente pulado naquele frame (sem erro).

### 5.2 Trajetória desenhava sempre em direção fixa (não seguia a câmera)

**Causa raiz:** o projeto usa o **Gameplay Camera System** (plugin novo da UE, não o SpringArm+Camera clássico) rodando em modo *standalone*. Nesse modo, `PlayerCameraManager` não reflete a rotação real da câmera — fica com um valor obsoleto/default.

**Correção:** a direção de mira passou a vir de `Pawn->GetController()->GetControlRotation()` (rotação de controle do jogador, atualizada pelo input de mouse), em vez de `PlayerCameraManager`.

*(Nota histórica: uma tentativa intermediária usou `Camera|GetEvaluatedCameraRotation` do componente `GameplayCamera`, mas o nó foi criado como efeito colateral de uma chamada de introspecção de API e foi descartado silenciosamente pelo compilador — por isso a solução final usa `GetControlRotation`, criado explicitamente.)*

### 5.3 Trajetória final "caía" em cima do próprio personagem ao mirar em certos ângulos

**Causa raiz:** o ponto-alvo (`EndPos`) era calculado como "posição da mão + direção da câmera × 700 unidades fixas", sem considerar geometria real. Mirando para baixo/de certas formas, esse ponto acabava caindo sobre o próprio personagem.

**Correção:** o `EndPos` passou a vir de um **trace real** a partir de `GetActorEyesViewPoint` (posição dos "olhos" do personagem), seguindo a direção da mira até 3000 unidades, ignorando o próprio personagem (`bIgnoreSelf`). Se o trace acha uma superfície, usa o ponto de impacto; senão, usa o alcance máximo. Aplicado tanto no preview (`BP_CharacterMaster`) quanto no arremesso real (`GA_Throw`).

---

## 6. Sauce se destruía em cima do personagem ao arremessar de verdade (LMB)

Três causas distintas, todas corrigidas:

### 6.1 Física não "grudava" a velocidade no mesmo frame

**Causa:** `SetSimulatePhysics(true)` seguido imediatamente de `SetPhysicsLinearVelocity` no mesmo frame não funciona de forma confiável no Unreal — o corpo físico ainda não está totalmente registrado na cena de física.

**Correção em `GA_Throw`:** inserido `DelayUntilNextTick` entre a ativação da física/desanexação e a aplicação da velocidade.

### 6.2 Colisão física sólida contra o próprio personagem

**Causa raiz:** o `StaticMesh` do `BP_SauceMaster` tinha `Object Type = WorldStatic` e resposta de colisão padrão (não sobrescrita) de **Block** para o canal **Pawn**. A cápsula do personagem (`Object Type = Pawn`, perfil `Pawn`, e com `bSimulatePhysics = true` por causa do componente de Physical Animation) também bloqueia `WorldStatic` por padrão. Resultado: os dois lados concordavam em bloquear um ao outro — assim que o item era destacado (ainda praticamente dentro da cápsula do personagem) e ganhava velocidade, a física resolvia a colisão sólida contra o próprio lançador instantaneamente, zerando a velocidade.

**Correção:** resposta de colisão do canal **Pawn** no `StaticMesh` do `BP_SauceMaster` alterada de `Block` para `Ignore` — tanto no **default da classe** (CDO) quanto nas **3 instâncias já posicionadas** no mapa `L_Sandbox` (que tinham override de colisão serializado por instância e não herdavam a mudança do default da classe automaticamente). A detecção de "pegar" o item continua funcionando normalmente via `CatchDetectionSphere` (componente separado, não afetado por essa mudança).

### 6.3 (Complementar) Ignorar colisão de movimento contra o lançador

- Adicionado `IgnoreActorWhenMoving(Item, Character, true)` em `GA_Throw` logo após o `DetachItem` (mitigação complementar; a causa principal foi a 6.2 acima).

---

## 6.5 `BP_GrabPistol` — de "empurrar" para "agarrar segurando o LMB"

**Antes:** `GA_GrabShot` fazia um multi-trace a partir do `PlayerCameraManager` e aplicava impulso (`AddImpulse`) nos objetos com `BPI_Grab`.

**Agora:**
- `GA_GrabShot` (`EventGraph`): apenas chama `TryGrab` e `EndAbility`.
- `GA_GrabShot::TryGrab` (nova função): só roda em quem controla localmente (`Ability|IsLocallyControlled`), ignora se `bGrabbing` já é true, exige `IsInputKeyDown(LeftMouseButton)` (evita re-grab fantasma por causa do `Delay 0.2` do `IA_Shoot`), mira com `GetActorEyesViewPoint` (ControlRotation), faz `LineTraceByChannel` (alcance `GrabRange`, ignorando personagem e item segurado), valida `BPI_Grab` + `IsSimulatingPhysics`, chama `PhysicsHandle.GrabComponentAtLocationWithRotation` no ponto de impacto, grava `GrabDistance` no pistola e seta `bGrabbing = true`.
- `BP_GrabPistol`: variáveis `GrabDistance`, `GrabRange` (2500) e `MinGrabDistance` (150) (categoria *Grab*; as duas últimas `Instance Editable`). `EventTick` habilitado (`bCanEverTick = true`): enquanto `bGrabbing`, atualiza a posição-alvo do `PhysicsHandle` (ver 6.6); se o componente sumir ou parar de simular, solta e zera `bGrabbing`.
- **Soltar:** já era tratado pelo `IA_Shoot Completed/Canceled` em `BP_CharacterMaster` (`PhysicsHandle.ReleaseComponent` + `bGrabbing = false`) — não foi alterado.
- **Limitações:** grab é local (não replica a física para outros clientes); `Delay 0.2` no `Triggered` do `IA_Shoot` (compartilhado com as outras abilities) causa ~0.2s de latência até o objeto ser agarrado; toques de LMB mais curtos que 0.2s nunca agarram.

---

## 6.6 Grab não funcionava com `BP_ItemMaster` + distância, debug e altura ajustáveis

### 6.6.1 Bug: o trace acertava a `Sphere` em vez da malha

**Causa raiz:** `TryGrab` faz o trace em `TraceTypeQuery1` (canal **Visibility**). No `BP_ItemMaster` (classe e instância `BP_ItemMaster_C_1` do `L_Sandbox`):
- `StaticMesh` simula física mas tem `Visibility = Ignore` → o trace atravessa a malha.
- `Sphere` (raio 100, `ObjectType = Interactable`, `QueryOnly`, não simula) não sobrescreve `Visibility` → usa o padrão **Block** → o trace acerta a esfera.

O ator implementa `BPI_Grab` (passava na checagem), mas `IsSimulatingPhysics(HitComponent)` dava `false` para a esfera, e o grab falhava em silêncio.

**Correção (`GA_GrabShot`):**
- Nova função `ResolveGrabComponent(HitActor, HitComponent) → GrabComponent`: se `HitComponent` simula física devolve ele mesmo; senão devolve o `StaticMeshComponent` do `HitActor` (`GetComponentByClass`).
- `TryGrab` chama a função após a checagem de `BPI_Grab`; `IsSimulatingPhysics`, `GetWorldRotation` e `GrabComponentAtLocationWithRotation` passaram a usar o componente resolvido.
- Colisão dos itens **não** foi alterada (a esfera continua servindo à detecção de pickup; `Visibility = Ignore` no mesh é mantido porque outros traces — arremesso/mira — usam esse canal).
- Se o ator não tiver `StaticMesh` e o `HitComponent` não simular, o resultado é nulo: só gera "Accessed None" e o grab aborta.

### 6.6.2 Distância do objeto pela roda do mouse

- Enquanto segura o objeto, a roda do mouse altera `GrabDistance` em passos de `GrabDistanceStep`, com clamp entre `MinGrabDistance` e `MaxGrabDistance`. Rolar para cima afasta; use um `GrabDistanceStep` negativo para inverter.
- Variáveis novas em `BP_GrabPistol` (categoria *Grab*, `Instance Editable`): `MaxGrabDistance` (padrão **2500**, igual ao `GrabRange`) e `GrabDistanceStep` (padrão **100**).
- A roda é lida direto com `GetInputAnalogKeyState(MouseWheelAxis)`, sem `InputAction` nem alteração no `IMC_Inputs` (criar action/mapping por ferramenta era arriscado). Consequência: não é remapeável.
- Se `MaxGrabDistance` for menor que `GrabRange`, um objeto agarrado mais longe que o máximo "pula" para dentro do limite no primeiro frame.

### 6.6.3 Linha de debug pistola → objeto

- Linha verde (`DrawDebugLine`, espessura 3, duração 0 = redesenhada todo frame) do mesh da pistola até a origem do componente agarrado.
- É `DrawDebugLine`: só aparece no editor e em builds de desenvolvimento. Para feedback no jogo final, trocar por Niagara beam/cabo.

### 6.6.4 Altura do ponto de grab ajustável

**Problema:** o objeto agarrado ficava baixo demais (o alvo do `PhysicsHandle` ficava na linha de visão, e o objeto é puxado pelo ponto onde foi agarrado, não pelo centro).

**Correção:** nova variável `GrabHeightOffset` em `BP_GrabPistol` (categoria *Grab*, `Instance Editable`, padrão **75**), somada em Z (mundo) ao alvo: `olhos + forward * GrabDistance + (0, 0, GrabHeightOffset)`. Positivo sobe, negativo desce. Ajustável em *Class Defaults* ou por instância no nível (funciona em PIE sem recompilar).

### 6.6.5 Refatoração no `BP_GrabPistol`

- Nova função `UpdateHeldObject(GrabbedComponent, Character)`: aplica a roda do mouse com clamp, chama `PhysicsHandle.SetTargetLocation` (com o offset de altura) e desenha a linha de debug.
- `EventTick`: o ramo "componente válido e simulando física" agora só chama `UpdateHeldObject`; o `SetTargetLocation` antigo e os nós de conta que ficaram órfãos foram removidos (a ferramenta de criação de nós não consegue criar soma de vetores, então a conta foi movida para a função, escrita via DSL).

---

## 6.7 Grind/Slide em rails (`BP_SplineGrindMaster`)

O `BP_SplineGrindMaster` só montava segmentos de `SplineMesh` ao longo da `Spline` (sem gameplay). O `BPI_Interact.GetGrindSpline` já existia, mas sem uso nem implementação; o Grind lê a `Spline` direto do rail.

### 6.7.1 Comportamento

- **Detecção e início:** o pawn precisa estar **armado** (ver 6.7.3), sem Grind ativo, fora do cooldown, caindo (`vz < -GrindMinFallSpeed`) e sobrepondo um `BP_SplineGrindMaster`. Com o centro da cápsula acima do trilho (−20) e a até `GrindMaxSnapDistance` do ponto mais próximo da spline, e com trilho à frente na direção do movimento (>0,02 de chave).
- **Direção:** sinal da velocidade projetada na tangente da spline; se `|vel·tangente| ≤ 50`, usa o vetor frontal da cápsula.
- **Percurso:** avança pelo **input key** da spline (`key += dir * GrindSpeed * dt / |tangente_mundo|`), sem usar `GetSplineLength`/distâncias (que neste UE 5.8 não são consistentes com a escala do ator). Funciona com rails escalados. Gravidade da cápsula desligada; velocidade = `(alvo − posição)/dt` limitada a `3 * GrindSpeed` (robusta a qualquer FPS); a cápsula gira para a tangente (velocidade angular em Z limitada a ±360°/s). O alvo fica `GrindHeightOffset` acima da spline.
- **Fim:** ao chegar à chave máxima/mínima, `EndGrind` religa a gravidade e o pawn sai com a inércia.
- **Saída por pulo:** um novo pressionamento de Jump durante o Grind chama `EndGrind` e o pulo original executa em seguida. Se o botão já estava segurado ao encaixar, precisa ser solto antes (`bJumpHeld`/`bGrindJumpReleased`). Enquanto o Grind está ativo, o corpo do `Jump` original é pulado.
- **Sem re-encaixe imediato:** ao sair (fim ou pulo), o pawn ignora aquele rail (`GrindIgnoreSpline`) até tocar o chão; sem isso o pulo (~130 u de altura) reencaixava em ~0,6 s. Há também `GrindCooldown`.

### 6.7.2 Implementação (`BPC_WheelchairMovement`)

- **Funções novas:** `TryStartGrind`, `StartGrind(Spline)`, `UpdateGrind`, `EndGrind`, `HandleGrindJump(bJump)`.
- **Variáveis novas (categoria *Grind*):** `bIsGrinding`, `bJumpHeld`, `bGrindJumpReleased`, `bGrindArmed`, `GrindSpline`, `GrindIgnoreSpline`, `GrindInputKey`, `GrindDirection`, `GrindResumeTime`. Editáveis por instância: `GrindSpeed` (900), `GrindHeightOffset` (190), `GrindMinFallSpeed` (100), `GrindCooldown` (0,5), `GrindMaxSnapDistance` (300).
- **Encaixe em `Movement`:** `Movement → Branch(bIsGrinding)`: verdadeiro → `UpdateGrind`; falso → `CheckIfFalling` → `TryStartGrind` → fluxo original. `CheckIfFalling` roda **antes** de `TryStartGrind` para o teste de chão ser atual.
- **Encaixe em `Jump`:** `Jump → HandleGrindJump(bIsJump) → Branch(bIsGrinding)`: falso → fluxo original. Na saída por pulo, `bIsFalling` é forçado a `true` para o pulo original passar na condição de "no chão" (o nome está invertido: `true` = no chão).
- **Padrões:** aplicados na classe (`Default__BPC_WheelchairMovement_C`).

### 6.7.3 Grind só depois de pular (fix do "encaixa em degrau")

**Problema:** o Grind disparava ao passar por um degrau/desnível no chão perto de um rail. A cápsula é um corpo físico e um degrau já produz `vz < −100`.

**Correção:** `bGrindArmed`. O `Jump` seta `true` depois de aplicar o impulso (só acontece quando o pulo realmente ocorre). `TryStartGrind` seta `false` quando o pawn está no chão e `vz ≤ 50` (o limite evita desarmar no 1º frame do pulo, quando o chão ainda é detectado) e exige `bGrindArmed` para iniciar. Cair de uma borda sem pular **não** faz Grind. A saída por pulo do Grind rearma, permitindo encaixar em outro rail.

### 6.7.4 Mudança no `BP_SplineGrindMaster`

Cada segmento de `SplineMesh` agora responde **Overlap** a `Pawn` e `PhysicsBody` (2 chamadas `SetCollisionResponseToChannel` no loop do `ConstructionScript`). Antes o rail (`Custom`, tipo `Interactable`, resto `Block`) bloqueava fisicamente a cápsula, que arrastava a ~190 u/s e nunca gerava overlap real (a detecção só funcionava por acaso via a esfera `WidgetRadius` herdada de `BP_InteractableMaster`, que fica na origem do ator). **Consequência:** o rail deixa de ser sólido para o jogador. `Visibility`/`Camera` continuam Ignore.

### 6.7.5 Validação (PIE)

- Queda sobre o rail: START, ~905 u/s ao longo do rail, END na ponta (chave 0,996), sem re-início.
- Pulo durante o Grind: sai no mesmo frame, sem re-encaixe até tocar o chão.
- Sem pular (pawn dentro do volume do rail): sem Grind. Pulando: Grind. `PrintString`s de debug usados nos testes foram removidos.
- **Não testado:** spline curva ou com vários segmentos, degrau real dirigindo, pulo normal fora do Grind em runtime (só leitura do grafo), multiplayer (o Grind roda onde o `Movement` roda, sem checagem de autoridade).
- **Nota de teste:** o editor em segundo plano estrangula o PIE a ~3 FPS (`bThrottleCPUWhenNotForeground`); para testes automatizados foi desligado só em memória e restaurado.

---

## 6.8 Multiplayer — bloco 1: movimento e RPCs (servidor autoritativo)

Contexto: análise de replicação (somente leitura) apontou que o projeto não estava pronto para multiplayer. Modelo adotado: **servidor autoritativo para física/habilidades; o cliente só envia input**. Este bloco cobre movimento e pulo (itens C3/C4 da análise). Interação/itens/habilidades (C1/C2/C4b), animações de mira/emote, grab/grind e itens médios **ainda não foram tratados**.

### 6.8.1 Diagnóstico (resumo)

- RPCs decodificados do `.uasset` (as ferramentas MCP não expõem flags): `ServerRPC_SendInput`, `Server_Jump`, `RPC_LocalMic`, `Server_SetSpeaking` = Server + Reliable; `Multicast_Jump` = Multicast **Unreliable**; `Rettach/Detach Player Multicast` = Multicast Reliable; `BPC_WheelchairMovement.ServerRPC_Movement` = Server **Unreliable**.
- `BP_CharacterMaster.EventTick` chamava `Movement` em todas as máquinas: o servidor executava `ServerRPC_Movement` localmente **e** recebia a RPC do cliente → `AddForce` em dobro; proxies simulados chamavam a RPC sem dono (warning `No owning connection ... ServerRPC_Movement will not be processed`).
- `IA_Move` (Triggered) e `IA_Jump` (Started + Triggered) enviavam RPC **Reliable a cada frame**.
- O AnimBP (`ABP_WheelChair`) lê `bIsJump`, `bIsFalling`, `ForwardInput` e `TurnInput` de `BPC_WheelchairMovement`, que não replicava (componente com `bReplicates = false`).

### 6.8.2 Mudanças

- **`BPC_WheelchairMovement.Movement`:** agora começa com `Switch Has Authority`; só o ramo *Authority* executa (Grind, `CheckIfFalling`, gravidade extra, damping, `ServerRPC_Movement`). Clientes não simulam movimento localmente (dependem da física replicada; a resposta ao input tem a latência de ida e volta).
- **Replicação do componente:** `SetIsReplicated(true)` no `BeginPlay` do componente (o template no `BP_CharacterMaster` gravava `bReplicates = false` e sobrepunha o padrão da classe; o CDO também foi marcado `true`). Variáveis `ForwardInput`, `TurnInput`, `bIsJump` e `bIsFalling` agora `Replicated`.
- **Pulo:** `IA_Jump` só chama `Server_Jump` no `Started` (o link do `Triggered` foi removido); `Server_Jump` chama `BPC_WheelchairMovement.Jump` direto no servidor, sem `Multicast_Jump` (o evento continua no grafo, sem uso). Resultado: 1 RPC no press e 1 no release.
- **`IA_Move` (asset):** trigger `Pulse` (`interval = 0.05`, `bTriggerOnStart = true`) → envio a 20 Hz em vez de por frame. `Canceled` agora também liga em `ServerRPC_SendInput(0,0)`, porque soltar a tecla entre pulsos gera `Canceled` (não `Completed`).
- **Não foi possível** (limite das ferramentas): mudar Reliable → Unreliable dos RPCs, nem inserir "enviar só se mudou" antes de `ServerRPC_SendInput` (o evento vive num grafo colapsado e não é chamável de função/EventGraph).

### 6.8.3 Validação (PIE em rede: servidor + 2 clientes)

- Espaço no *Client 2* → `Server_Jump` executa **no servidor**, uma vez no press e uma no release; nenhuma execução nos clientes.
- Os warnings `No owning connection ... ServerRPC_Movement` deixaram de aparecer.
- No cliente: `BPC_WheelchairMovement.bReplicates = true` e `bIsFalling` chega replicado.
- **Não testado:** movimento com tecla segurada e animação de rodas em clientes (as ferramentas só enviam press+release instantâneo); Grind em rede.
- Erro de runtime que permanece (não tratado neste bloco): `BP_WWController.BeginPlay` cria o HUD (`CreateWidget`/`AddToViewport`) também no servidor → `Accessed None` (item M1 da análise).

### 6.8.4 Limpeza: nós órfãos deixados por `write_graph_dsl`

Reescrever um grafo via DSL **não apaga os nós antigos**: eles ficam desconectados do fluxo (não executam), mas inflam o `.uasset` e mantêm strings antigas. Encontrado: `StartGrind` 359 órfãos (de 416 nós), `HandleGrindJump` 52 (de 62), `BP_GrabPistol.UpdateHeldObject` 14 (de 39), incluindo `PrintString` de debug — o commit `cc6a73f` já continha strings `GRINDDBG`. Os órfãos foram apagados por script (nós inalcançáveis por execução e que não são dependência de dados de nó vivo); `BPC_WheelchairMovement.uasset` caiu de ~1,5 MB para ~655 KB e não há mais strings de debug. **Ao usar `write_graph_dsl`, conferir órfãos depois.**

---

## 6.9 Multiplayer — bloco 2: replicação completa (interação, itens, habilidades, estados)

Modelo mantido: **servidor autoritativo**; o cliente só envia intenção + mira. Validado em PIE com servidor dedicado + 2 clientes, comparando o estado nos 3 mundos.

### 6.9.1 Canal cliente → servidor via GAS

As ferramentas MCP não criam RPCs nem alteram flags (e `write_graph_dsl` no `EventGraph` de `BP_CharacterMaster` **apaga o grafo colapsado "Inputs"** — testado numa cópia; esse grafo só foi editado nó a nó). Solução:
- **`GA_ClientRequest`** (novo): `LocalPredicted`, `InstancedPerActor`, trigger por Gameplay Event `Event.Request` (tags filhas também disparam). No servidor chama `BP_CharacterMaster.HandleClientRequest(EventData)`, que despacha pela tag: `Event.Request.AnimState`, `.Horn`, `.GrabUpdate`, `.GrabRelease`.
- **`SendClientRequest(Tag, Magnitude, Start, End)`**: monta o `GameplayEventData` com um `HitResult` em `TargetData` (`TraceStart`/`Location`); o GAS leva os dados ao servidor.
- **Tiro:** `HandleShoot` envia `Event.Ability.Shoot` com a mira da câmera (origem + direção). `GA_SauceShot`, `GA_Throw` e `GA_GrabShot` passaram a ser disparadas por essa tag (triggers configurados) e só agem com autoridade, refazendo o trace no servidor com o alcance original de cada uma (2500 / 3000 / `GrabRange`).
- No ramo do cliente as abilities usam **`EndAbilityLocally`**: `EndAbility` replicava o fim ao servidor e matava a lógica latente do `GA_Throw` (`DelayUntilNextTick`), que soltava o sauce sem arremessar.

### 6.9.2 Input (`BP_CharacterMaster`)

Handlers de `IA_Move`, `IA_Aim`, `IA_Emote`, `IA_Shoot` e `IA_Horn` saíram do grafo colapsado e foram recriados no `EventGraph`, chamando funções novas: `HandleMoveInput` (quantiza 0,05 e só envia `ServerRPC_SendInput` quando muda, com reenvio a cada 0,25 s enquanto ≠ 0), `HandleAim`, `HandleEmote`, `HandleShoot`, `HandleShootReleased`, `HandleHorn`. Jump, Interact, Look, Sprint e os eventos RPC continuam no grafo colapsado, intactos. As flags dos 8 RPCs originais foram conferidas no `.uasset` depois das edições.

### 6.9.3 Estados e animação

- `AnimationChooser` (RepNotify) agora é definido no servidor (`RequestAnimState` → `ApplyAnimState`), com previsão local no dono; `BoneAnim` passou a `Replicated`. O servidor aplica `Idle` no `BeginPlay`.
- **Armadilha do enum:** em `E_PlayerChooser` os **valores numéricos não batem com os nomes** `NewEnumeratorN` (o valor 0 é "None"). Os chamadores usam literais do enum (`NewEnumerator0` Idle, `1` Smoking, `4` Aim, `6` Throw) em vez de números.
- O gate do LMB aceita `Aim` **ou** `Throw` — antes só `Aim`, então o `GA_Throw` nunca era ativado pelo input.
- Buzina: `HornCount` (RepNotify); `OnRep_HornCount` toca o som nos outros jogadores.
- Lip-sync: o Tick agora envia `Server_SetSpeaking(true)` também (antes só enviava `false`).

### 6.9.4 Itens, projéteis e splat

- `BP_ProjectileMaster`: `bReplicates` + `bReplicateMovement`; o hit (spawn do splat + destroy) só com autoridade.
- `BP_SplatSauce`: `bReplicates`; `SplatIndex` (Replicated) sorteado no servidor para a variação ser igual em todos; lifespan e atrito (`bOnSplat`) só com autoridade.
- `BP_SauceMaster`: catch e pouso (splat + destroy) só com autoridade; removido o `PrintString` por tick.
- `BP_ItemMaster.OnRep_bHeld`: nos clientes anexa o item ao socket da mão do `Owner` (e desanexa ao largar). Corrige a condição de corrida em que a `AttachmentReplication` chegava com o item ainda simulando física e os clientes o viam flutuando onde estava.
- `BPC_InteractionDetector`: `AttachItem` faz `ClearAbility` da ability anterior antes de conceder a nova (handle em `ShotAbilityHandle`) — sem duplicatas e sem cancelar uma ability em execução; `DetachItem` limpa `EquippedItemInfo` e o impulso ao largar passou a ser só para cima (antes somava a posição no mundo ao vetor).

### 6.9.5 Grab (`BP_GrabPistol` / `GA_GrabShot`)

Grab feito no servidor (`PhysicsHandle`); `bGrabbing` e `GrabDistance` replicados. O dono calcula o alvo (câmera + roda do mouse) e envia `Event.Request.GrabUpdate` a no máximo 20 Hz e só se o alvo mexer > 1 cm; o servidor aplica e solta sozinho se o objeto sumir/parar de simular (`ServerValidateGrab`). Removido o `IsInputKeyDown(LeftMouseButton)` fixo.

### 6.9.6 Movimento, crash e framework

- `BPC_WheelchairMovement`: `ServerRPC_Movement` usa o `CurrentState` **do personagem** (não andava mais o valor do componente, que nunca mudava) → não dirige tombado; inputs com clamp −1..1 no servidor. Componentes com `bReplicates = true` no template.
- Crash: `OnComponentHit` só com autoridade; `RettachPlayerMulticast` só é disparado pelo servidor (antes os clientes o executavam em dobro); camera shake só no jogador local; `SetRelativeTransform` do reattach com `Teleport`.
- `SetupLocalPlayer` (via `ReceiveControllerChanged`): `PlayerControllerReference` também no servidor, mapping context no cliente mesmo se o `BeginPlay` rodar antes da posse, microfone só no pawn local.
- `BP_WWController`: HUD só com `IsLocalPlayerController` (o erro `Accessed None` no servidor sumiu).
- `BP_InteractableMaster`: prompt mostra/esconde só para o pawn local (o `FlipFlop` era alternado por qualquer jogador).
- `DefaultEngine.ini`: `GlobalDefaultGameMode` apontava para `/Game/Main/Core/BP_WWGameMode` (inexistente) → `/Game/Main/Core/Framework/BP_WWGameMode`.

### 6.9.7 Validação (PIE: servidor dedicado + 2 clientes)

Input simulado por um harness temporário no `BP_WWController` (removido depois; `PressKey` não aciona o Enhanced Input). Conferido nos 3 mundos: estado inicial, emote on/off (também a partir do cliente 2), buzina, mira, pickup/largar/troca de arma (sem abilities duplicadas), grab (objeto na mesma posição para todos) e soltar, tiro de molho (splat na mesma posição e índice), arremesso completo (arco → pouso → splat → destroy nos clientes), movimento (posição idêntica; keep-alive ativo) e crash → recuperação. Log sem erros novos; todos os 25 Blueprints compilam com warnings-as-errors.

**Não testado ao vivo:** lip-sync (precisa de microfone), atrito do splat (o pawn passou rápido demais).

---

## 7. Débito técnico / pendências

- ~~Prints de debug temporários em `GA_Throw` e `BP_SauceMaster`~~ — removidos no bloco 2 (6.9).
- **Crash:** qualquer contato com ator de tag `Obstacle` derruba (limiar `velocidade > 0 * 0.2`); encostado numa parede, tomba de novo a cada 3 s. Comportamento anterior, não alterado.
- **RPCs Reliable:** `ServerRPC_SendInput` continua Reliable (as ferramentas não mudam flags); o envio agora é só na mudança/keep-alive. Trocar para Unreliable no editor é opcional.
- **Código morto mantido:** `RPC_LocalMic`, `Multicast_Jump`, `JawOpenAmount` replicado sem uso, função `GA_GrabShot::TryGrab` antiga.
- **Sprint:** o branch de `IA_Sprint` tem condição `AND(false, …)` — não faz nada (anterior).
- **Parâmetros de arremesso hardcoded** (não expostos para ajuste manual): alcance do trace (3000), altura do arco (`ArcParam = 0.35`), gravidade do arremesso (`OverrideGravityZ = -1500`), tempo de desenho do debug (`DrawDebugTime = 2.0`), limiar de velocidade para considerar "pousado" (`5.0`). Considerar expor como variáveis `Instance Editable` em `GA_Throw`/`BP_SauceMaster` se for necessário balancear o gameplay.
- **`.gitignore`**: os padrões `Saved/`, `Intermediate/`, `DerivedDataCache/`, `Binaries/` e `Build/` foram prefixados com `**/` para funcionar com o projeto aninhado em `WheelyWiener_V01/` (ver `PROJECT_STATUS.md`, item 4). `.idea/` ainda não é ignorado.
- **Grab — pendências abaixo foram tratadas no bloco 2 (6.9.4/6.9.5), exceto a leitura direta da roda do mouse:**
  - `BPC_InteractionDetector::DetachItem` não faz `ReleaseComponent`, não zera `bGrabbing`, não limpa o `Owner` nem faz `ClearAbility`; largar/trocar a pistola segurando um objeto deixa o estado inconsistente.
  - `AttachItem` chama `GiveAbility` a cada pickup sem remover a anterior → abilities duplicadas ao pegar/largar várias vezes.
  - O branch do `IA_Shoot` em `BP_CharacterMaster` testa `AnimationChooser == NewEnumerator4`, mas o item 4.4 acima diz que a condição foi trocada por `true`; não foi confirmado o que é `NewEnumerator4` nem se equipar a `BP_GrabPistol` seta esse valor.
  - `TryGrab` usa `IsInputKeyDown(LeftMouseButton)` fixo (quebra com rebind/gamepad); a roda do mouse também é lida direto.
  - Grab e linha de debug são só locais (`ItemMaster` tem `bReplicateMovement = false`).
- **Grind — observações:**
  - O corpo de `BPC_WheelchairMovement::Jump` roda também no *release* do botão (e sem guarda de `bIsJump`), repetindo o impulso de 800; comportamento anterior, não alterado.
  - Rail com escala não uniforme não foi testado; o `GrindHeightOffset` padrão (190) assume o mesh `SM_BoxCentered` (100 de altura) e cápsula de meia altura 90.
  - Rails deixaram de ser sólidos para o Pawn/PhysicsBody (ver 6.7.4).
  - O `.uasset` do `BPC_WheelchairMovement` cresceu (~240 KB → ~655 KB) por causa dos nós gerados via DSL (depois da limpeza de órfãos, ver 6.8.4).
- **Multiplayer — pendências da análise: todas tratadas no bloco 2 (6.9)**, exceto Grind (roda só no servidor via `Movement`; não testado em rede). Lista original: interação/`AttachItem` e `HeldItem` só no cliente que apertou F, `GiveAbility` chamado no cliente (inválido no GAS), projéteis/splat/`bIsThrown` não replicados e `DestroyActor`/`SpawnActor` em todas as máquinas, `AnimationChooser` (mira/emote) setado só localmente, grab e grind sem servidor, `PlayerControllerReference` só no dono (usado por `GA_SauceShot`), `OnComponentHit`→`DetachPlayerMulticast` em todas as máquinas, HUD sem `IsLocalController`, prompt de `BP_InteractableMaster` sem checar jogador local.
- **Interação (F) com `BP_ItemMaster` — diagnóstico (sem alteração de código):** a instância `BP_ItemMaster_C_1` do `L_Sandbox` tem `ItemInfo = None`; `AttachItem` lê `ShotAbility` e `Socket` do `ItemInfoReference` e gera `Accessed None` (nós `Branch` e `Attach Actor To Component`), então o item não fica na mão. Além disso, `InteractDetect` não valida o resultado do trace (F em parede/vazio chama `AttachItem` com ator inválido e solta o item segurado) e o trace sai da raiz do `PlayerCameraManager` (~800 u atrás do pawn, 1200 de comprimento, na altura da câmera).

---

## Arquivos modificados/criados nesta sessão

| Arquivo | Tipo de mudança |
|---|---|
| `Content/Main/Core/Components/BPC_WheelchairMovement` | Fix gravidade extra + damping angular; **Grind/Slide** (funções `TryStartGrind`/`StartGrind`/`UpdateGrind`/`EndGrind`/`HandleGrindJump`, variáveis *Grind*, gate `bGrindArmed`, ramos em `Movement` e `Jump`) |
| `Content/Main/Core/Interactables/BP_SplineGrindMaster` | Segmentos do rail com resposta Overlap a `Pawn`/`PhysicsBody` |
| `Content/Main/Core/Components/BPC_WheelchairMovement` (multiplayer) | `Movement` só com autoridade, `SetIsReplicated(true)`, 4 variáveis replicadas, limpeza de nós órfãos |
| `Content/Main/Core/Player/BP_CharacterMaster` (multiplayer) | `Server_Jump` chama `Jump` direto (sem `Multicast_Jump`), `IA_Jump` só no `Started`, `IA_Move Canceled` → `SendInput(0,0)` |
| `Content/Main/Core/Inputs/Actions/IA_Move` | Trigger `Pulse` (0,05 s) |
| `Content/Main/Core/Items/BP_GrabPistol` | Limpeza de nós órfãos em `UpdateHeldObject` |
| `Content/Main/Core/Components/BPC_InteractionDetector` | Fix colisão da esfera do item + guarda `IsValidClass` |
| `Content/Main/Core/Abilities/GA_Throw` | **Novo** — ability de arremesso |
| `Content/Main/Core/Abilities/GA_Catch` | **Novo** — ability de captura |
| `Content/Main/Core/Items/BP_SauceMaster` | Variáveis `bIsThrown`/`ThrowerReference`, `CatchDetectionSphere`, lógica de Catch/Miss, fix de colisão Pawn=Ignore |
| `Content/Main/Core/DataAssets/DA_SimpleSauce` | `ShotAbility = GA_Throw` |
| `Content/Main/Core/Player/BP_CharacterMaster` | Concessão de `GA_Catch` no BeginPlay, fix do gate de input LMB, preview contínuo de trajetória no Tick |
| `Content/Main/Core/Abilities/GA_GrabShot` | `TryGrab` + função `ResolveGrabComponent` (fix do grab em `BP_ItemMaster`) |
| `Content/Main/Core/Items/BP_GrabPistol` | Função `UpdateHeldObject`; variáveis `MaxGrabDistance`, `GrabDistanceStep`, `GrabHeightOffset`; roda do mouse, linha de debug, offset de altura |
| `.gitignore` | Padrões `Saved/`, `Intermediate/`, `Binaries/`, `Build/`, `DerivedDataCache/` com `**/` |
| `Content/Main/Maps/Sandbox/L_Sandbox` | Fix de colisão Pawn=Ignore nas 3 instâncias de `BP_SauceMaster` já posicionadas |
| `Content/Main/Core/Abilities/GA_ClientRequest` | **Novo** (bloco 2) — canal cliente → servidor |
| `Content/Main/Core/Abilities/GA_SauceShot`, `GA_Throw`, `GA_GrabShot` | Bloco 2 — trigger `Event.Ability.Shoot`, lógica só com autoridade, `EndAbilityLocally` no cliente |
| `Content/Main/Core/Player/BP_CharacterMaster` (bloco 2) | 16 funções novas, handlers de input no `EventGraph`, `HornCount`, `BoneAnim`/`bGrabbing` replicados, gates de autoridade no crash |
| `Content/Main/Core/Items/BP_ItemMaster`, `BP_ProjectileMaster`, `BP_SplatSauce`, `BP_SauceMaster`, `BP_GrabPistol` | Bloco 2 — replicação e autoridade |
| `Content/Main/Core/Components/BPC_InteractionDetector`, `BPC_WheelchairMovement` | Bloco 2 — `ClearAbility`, `EquippedItemInfo`, estado do personagem, clamp |
| `Content/Main/Core/Framework/BP_WWController`, `Interactables/BP_InteractableMaster` | Bloco 2 — HUD e prompt só locais |
| `Config/DefaultEngine.ini`, `Config/DefaultGameplayTags.ini` | Caminho do GameMode; tags `Event.Request.*` e `Event.Ability.Shoot` |

---

*Documento gerado automaticamente a partir do histórico de alterações feitas via MCP nesta sessão de trabalho.*
