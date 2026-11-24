# Comparador de espessuras em 3D: guia para a TI

Guia do arquivo `index.html` deste repositório (o mesmo `espessura-lente-3d.html` entregue à TI): o que ele faz, como subir, como a conta funciona e onde mexer para personalizar.

## 1. Resumo

- **O que é:** uma página que compara duas lentes lado a lado, em 3D, em dois índices de refração. Substitui a tela "Comparar Espessuras" do CDL.
- **Formato:** um arquivo HTML único, com CSS e JavaScript dentro. Cerca de 56 KB (20 KB com gzip).
- **Dependências:** nenhuma biblioteca. O 3D é WebGL puro. A única coisa buscada de fora é a fonte Geist, no Google Fonts.
- **Dados:** nada é salvo nem enviado. A receita digitada fica só na memória da página e some ao recarregar.
- **Dois modos:**
  - **Modo simples:** grau genérico em um controle deslizante, usando os percentuais do catálogo ("40% mais fina").
  - **Medida personalizada:** receita e armação do cliente, com espessura em cinco pontos, peso e diâmetro da lente bruta.

## 2. Como subir

1. Hospedar o arquivo como página estática (por exemplo, na pasta pública do Gestor).
2. Abrir dentro do CDL por `iframe`, ou por link ou botão.
3. Dar ao `iframe` uma altura de pelo menos 540 px. A página ocupa 100% da altura que receber.

Pontos de atenção:

| Item | Detalhe |
|---|---|
| CSP | Se o site tiver Content Security Policy, liberar `fonts.googleapis.com` (style) e `fonts.gstatic.com` (font). Se ficar bloqueado, a página usa a fonte do sistema e funciona igual. |
| Script inline | O JavaScript está dentro do HTML. Uma CSP que proíba script inline bloqueia a página. Nesse caso, extrair o conteúdo da tag `<script>` para um arquivo `.js`. |
| WebGL | Necessário para o 3D. Sem WebGL, a página mostra o perfil das lentes em 2D e esconde os botões "Mapa de espessura" e "Medidas". |
| Movimento reduzido | Se o aparelho estiver com "reduzir movimento" ligado, as transições animadas são desligadas. |

## 3. Como o arquivo está organizado

O script é uma função única, dividida em blocos com comentários do tipo `/* ---------- nome ---------- */`. Na ordem:

| Bloco | O que tem |
|---|---|
| Início | `IDX` (índices), `S` (o que o usuário escolheu), `cur` (valores animados) |
| óptica (simplificada) | Conta do modo simples: `baseCurve`, `shape`, `tmax150`, `lens` |
| medida personalizada | Tabelas e conta do modo personalizado: `CMIN`, `DENS`, `EMIN`, `CORR`, `PRISMA`, `CU`, `shapeF`, `contour`, `custom` |
| geometria | `build`: monta a malha 3D de uma lente |
| matrizes, WebGL | Câmera e shaders (o vidro, o corte e o mapa de espessura) |
| sobreposição | Eixo tracejado, marcador vermelho e etiquetas de medida (SVG e HTML por cima do 3D) |
| `render` | Desenha as duas lentes e posiciona as etiquetas |
| interface | `ui` atualiza textos e botões. `layout` calcula posições |
| laço | `step` anima os valores e só roda enquanto algo está mudando |
| formulário | Abrir, validar e aplicar a medida personalizada |

Fluxo geral: um clique altera `S` ou `CU`, chama `ui()`, que chama `kick()`. O laço `step` leva `cur` até o alvo, reconstrói a malha (`rebuild`) e desenha (`render`). Quando tudo chega ao alvo, o laço para.

## 4. Modo simples

Entradas: dois índices (setas do topo), miopia ou hipermetropia, grau de 0,50 a 8,00.

Como calcula:

1. Calcula a lente padrão 1.50 para o grau escolhido, com 50 mm de diâmetro (`HR = 25`, o raio).
2. A maior espessura da lente padrão sai de `tmax150`: borda na miopia, centro na hipermetropia.
3. Para os outros índices, aplica o percentual da lista `IDX` (campo `pct`) sobre essa espessura.

Ou seja, **no modo simples o "quanto mais fina" vem do catálogo**, não da física. Para mudar um percentual, basta editar `IDX`.

A vista "No aro" recorta a lente em um aro genérico de 48 × 35 mm (função `Rf`, largura em `AW = 24`).

## 5. Medida personalizada

### 5.1 Entradas e limites aceitos

| Campo | Limite | Observação |
|---|---|---|
| Esférico | −14,00 a +10,00 | |
| Cilíndrico | até 6,00 | Cilindro positivo é transposto para negativo automaticamente |
| Eixo | 0 a 180 | |
| Adição | 0,50 a 4,00 | Só no multifocal |
| DNP | 20 a 40 mm | Medida de um olho só |
| Altura | 5 mm até (aro vertical − 3) | |
| Aro horizontal | 30 a 70 mm | |
| Aro vertical | 20 a 60 mm | |
| Ponte | 8 a 30 mm | |
| Diagonal maior | 30 a 85 mm | Usada só no diâmetro da lente bruta |

Os limites e as mensagens de erro ficam na função `formCheck`.

### 5.2 Como calcula (função `custom`)

1. **Recorte do aro.** `contour` descobre a distância do centro óptico até a borda do aro em 120 direções. O centro óptico é deslocado dentro do aro:
   - na horizontal, pela descentração: `(aro horizontal + ponte) ÷ 2 − DNP`
   - na vertical, pela altura medida
2. **Frente da lente.** Esférica, com curva base escolhida por `baseCurve` conforme o grau.
3. **Costas da lente.** Tóricas: a curva muda em cada direção conforme o cilindro e o eixo (`r2f`).
4. **Espessura em cada ponto.** Diferença entre a curvatura das costas e a da frente (`hf`), somada à espessura do centro.
5. **Espessura do centro.** É a maior entre:
   - o mínimo do material (`CMIN`)
   - o necessário para a borda mais fina não ficar abaixo do mínimo do tipo de aro (`EMIN`)
6. **Peso.** Volume da lente recortada (soma numérica da espessura em toda a área) × densidade (`DENS`).
7. **Diâmetro mínimo da lente bruta.** `diagonal maior + 2 × descentração + 2 mm`.
8. **"% mais fina".** Compara a maior espessura do índice escolhido com a do 1.50, para a mesma receita e o mesmo aro.

As cinco etiquetas mostram centro, nasal, temporal, superior e inferior. O resumo "Borda máx." é a maior espessura em todo o contorno, que pode ficar em um canto e ser maior que as quatro etiquetas.

### 5.3 Multifocal

Com "Multifocal" ligado, a adição entra assim:

- **Corredor (`CORR`, 14 mm).** A partir da altura medida, descendo, o grau ganha a adição aos poucos, até o valor cheio no fim do corredor. Dali para baixo fica constante.
- **Prisma de equilíbrio (`PRISMA`, 2/3).** Só é aplicado quando a adição deixa o topo mais grosso que a base. O valor é o suficiente para igualar os dois, limitado a 2/3 da adição.

Efeito típico: na negativa, a borda de baixo afina. Na positiva, o centro engrossa um pouco.

Simplificação conhecida: a adição vale em toda a largura da parte de baixo. Numa progressiva real ela se concentra no meio, então os cantos de baixo podem sair um pouco finos demais.

## 6. Tabelas para personalizar

Todas ficam juntas, no começo do bloco "medida personalizada".

| Variável | O que é | Valores atuais |
|---|---|---|
| `IDX` | Índices, nome do material e percentual do catálogo (modo simples) | 1.50, 1.53, 1.59, 1.60, 1.67, 1.74 |
| `CMIN` | Espessura mínima no centro, em mm | 2,0 / 1,8 / 1,5 / 1,5 / 1,3 / 1,2 |
| `DENS` | Densidade do material, em g/cm³ | 1,32 / 1,11 / 1,20 / 1,30 / 1,36 / 1,47 |
| `EMIN` | Borda mínima por tipo de aro, em mm | fechado 1,0 / nylon 1,8 / três peças 2,0 |
| `CORR` | Corredor do multifocal, em mm | 14 |
| `PRISMA` | Limite do prisma de equilíbrio, como fração da adição | 2/3 (use 0 para desligar) |

**Regra importante:** `CMIN` e `DENS` seguem a mesma ordem de `IDX`. Ao incluir ou tirar um índice em `IDX`, faça o mesmo nas duas, na mesma posição.

Esses valores são típicos de mercado, definidos sem consulta ao laboratório. São os primeiros candidatos a ajuste.

## 7. Como fazer as alterações mais comuns

### Mudar um percentual ou o nome de um material

Editar a linha `IDX`:

```js
var IDX=[{n:1.50,mat:'CR-39',pct:0},{n:1.53,mat:'CR-39',pct:10}, ...];
```

`n` é o índice, `mat` é o texto abaixo do número e `pct` é o "% mais fina" do modo simples. O catálogo atual chama o 1.53 de "CR-39". O usual para 1.53 é Trivex, e a densidade em `DENS` já é a do Trivex.

### Mudar os índices que abrem selecionados

Em `S`, os campos `a` (lado esquerdo) e `b` (lado direito) são a posição em `IDX`, começando em 0:

```js
var S={a:0,b:5, ...};   // 1.50 contra 1.74
```

### Mudar os valores que já vêm preenchidos no formulário

Editar o objeto `CU`: `od` e `oe` (receita de cada olho), `mha`, `mva`, `ponte`, `dma`, `mod` (formato) e `tipo` (tipo de aro).

### Incluir um formato de aro

1. Acrescentar a opção no `<select id="f-mod">`, com um `value` novo.
2. Acrescentar o desenho em `shapeF`. A função recebe a posição dentro da caixa do aro (`zt` de −1, ponta nasal, a 1, ponta temporal; `yn` de −1, base, a 1, topo) e devolve um número menor que 1 para pontos dentro do aro.

Os formatos atuais são superelipses: `p` maior deixa o aro mais quadrado, `p = 2` é uma elipse.

### Incluir um tipo de aro

1. Acrescentar a opção no `<select id="f-tipo">`.
2. Acrescentar a borda mínima em `EMIN`, com a mesma chave do `value`.

### Mudar o aviso de simulação

É o parágrafo com `id="sim"`:

```html
<p class="pill sim" id="sim" role="note" hidden>Simulação: a lente pronta pode ter espessura e peso diferentes.</p>
```

As cores ficam na regra `.sim` do CSS.

### Mudar a assinatura e a explicação da conta

- **Assinatura.** "Feito por Irving Arruda" aparece em dois lugares: no rodapé do cartão de baixo (`<div class="foot">`) e no fim da janela de explicação (`<p class="sig">`).
- **Contato.** Tocar no nome, no rodapé, abre um cartão pequeno com e-mail e telefone (`id="contact"`). Os mesmos dados aparecem no fim da janela de explicação. Para trocar, editar os dois lugares, incluindo os endereços `mailto:` e `tel:`.
- **Link "Como a conta é feita".** É o botão `id="info"`, no mesmo rodapé. Abre a janela `id="help"`.
- **Texto da explicação.** Fica direto no HTML, dentro de `id="help"`: o aviso amarelo (`class="warn"`), as fórmulas (`class="fx"`) e a lista do que a conta não considera.
- **Valores da explicação.** A tabela de materiais, as bordas mínimas, o corredor, o limite do prisma e os percentuais do catálogo são preenchidos pela função `openHelp` a partir de `IDX`, `CMIN`, `DENS`, `EMIN`, `CORR` e `PRISMA`. Quem mudar essas tabelas não precisa mexer no texto.
- **Atenção:** as fórmulas escritas na explicação são texto fixo. Se a conta em `custom` for alterada, o texto precisa ser revisado à mão.

### Mudar outros textos

- Textos fixos (botões, rótulos do formulário): direto no HTML.
- Textos que mudam com o estado (resumos, texto pequeno do rodapé): na função `ui`.
- Mensagens de erro do formulário: na função `formCheck`.

### Mudar cores

- Interface: variáveis no `:root` do CSS (`--stage`, `--card`, `--ink` e outras).
- Vidro da lente: constantes `GLASS0`, `GLASS1`, `EDGE0`, `EDGE1` e `CAP`, antes da função `render`. São cores em RGB de 0 a 1.
- Mapa de espessura: as três cores (verde, amarelo, vermelho) estão no shader, na linha que começa com `vec3 hc=`, e repetidas no CSS em `.rampbar` e `.btn.heat .sw`.

### Abrir já com a receita do cliente (integração com o Gestor)

Hoje a página não lê parâmetros. Se o Gestor for gerar a página já com os dados do pedido, dá para preencher `CU` e ligar o modo personalizado logo antes da última linha do script (`ui();`):

```js
CU.eye='oe';CU.multi=true;
CU.oe={esf:-3.25,cil:-0.5,eixo:90,add:2.25,dnp:32,alt:21};
CU.mha=54;CU.mva=40;CU.ponte=17;CU.dma=58;CU.mod='ret';CU.tipo='nylon';
var c0=contour(CU);
if(c0){CU.cont=c0.cont;CU.RX=c0.RX;CU.dec=c0.dec;CU.cy=c0.cy;CU.cz=c0.cz;CU.on=true;S.view='frame';S.meas=true}
```

Esse trecho foi testado. Ele **não passa pela validação** do formulário, então os valores precisam chegar dentro dos limites da seção 5.1 e com cilindro negativo.

## 8. Desempenho

- A página só redesenha quando algo muda (clique, arrasto, animação). Parada, não gasta bateria.
- A resolução do desenho é limitada a 2× a da tela (`dpr`).
- Cada lente tem poucos milhares de triângulos. A malha é refeita só durante as transições.

## 9. Limitações conhecidas

1. **É uma estimativa.** As suposições de curva base, espessura mínima e densidade são típicas, sem confirmação do laboratório.
2. **Sem validação com lente pronta.** Recomendação: medir a borda de 3 ou 4 lentes já entregues (pelo menos uma multifocal) e comparar com a simulação. Se houver diferença constante, ajustar `CMIN` e `EMIN`.
3. **Formato do aro é aproximado.** São 9 desenhos genéricos, não o traçado real da armação.
4. **Multifocal simplificado.** Ver a seção 5.3.
5. **Sem tratamentos.** Antirreflexo e outras camadas não entram na espessura.
6. **Teste em aparelho.** A página foi testada em navegador de teste, em tamanho de tablet e de celular. Falta testar o toque em um tablet de loja.

## 10. Conferência rápida depois de mexer

1. Abrir a página: as duas lentes aparecem cortadas ao meio, 1.50 contra 1.74.
2. Modo simples, miopia −4,00: resumos "Borda ≈ 4,6 mm" e "Borda ≈ 2,8 mm".
3. "Medida personalizada" com os valores padrão e "Simular": centro 2,0, nasal 3,0, temporal 3,9, superior 2,8, inferior 3,5 mm, e 6,0 g no 1.50.
4. Mesma receita em "Multifocal" com adição 2,00: inferior cai para 3,0 mm.
5. Aviso amarelo de simulação visível só no modo personalizado. "Como a conta é feita" abre a explicação, com a tabela de materiais preenchida.
6. "Modo simples" volta à tela inicial.

Se os números dos itens 2 a 4 mudarem sem que as tabelas tenham sido alteradas, alguma coisa na conta foi afetada.
