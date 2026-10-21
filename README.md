# Indicador RGB por Camada — Tergo Sofle rev1 (RP2040)

Proposta de modificação comunitária para o firmware da **Tergo Teclados** (baseada na tag oficial `tergo-sofle-v1.08`), adicionando indicação visual por cor no underglow conforme a camada ativa.

> [!WARNING]
> **Status do Pacote: Proposta para Revisão Técnica da Tergo.**  
> Esta modificação foi testada fisicamente com sucesso na unidade do autor em 20/10/2026, mas **não é um firmware oficial da Tergo Teclados**. Antes de qualquer teste ou instalação, leia atentamente as instruções de backup e recuperação abaixo.

---

## 🎯 O que esta modificação faz

1. **Iluminação por Camada:** O underglow muda de cor conforme a camada ativa temporária (`layer_state`); quando nenhuma camada temporária está ativa, consulta o modo base (`default_layer_state`).
2. **Independência de Layout no Vial:** O patch não modifica o mapeamento de teclas de fábrica. O usuário pode continuar alterando qualquer tecla pelo aplicativo Vial sem perder a indicação visual de camadas.
3. **Limite de Brilho no Código:** O valor de brilho repassado ao cálculo da cor respeita a constante `#define RGB_MATRIX_MAXIMUM_BRIGHTNESS 110` definida no `config.h` da fabricante.
4. **OLED Inalterado:** O arquivo `oled.c` permanece 100% idêntico ao original de fábrica da Tergo.

---

## 🎨 Paleta de Cores Implementada

| Camada | Nome Padrão Tergo | Cor do Underglow | Valores HSV |
|:---:|---|:---:|:---:|
| **L0** | `_BASIC` | ⚪ **Branco** | Saturação = 0 |
| **L1** | `_ADEPT` | 🟠 **Laranja** | H = 24, S = 255 |
| **L2** | `_CUSTOM` | 🔵 **Ciano** | H = 128, S = 255 |
| **L3** | `_SYMB_AND_NAV` | 🔷 **Azul** | H = 170, S = 255 |
| **L4** | `_ADJUST` | 🔴 **Vermelho** | H = 0, S = 255 |
| **L5** | `_MEDIA` | 🟣 **Roxo** | H = 195, S = 255 |
| **L6** | `_NUMPAD` | 🟢 **Verde** | H = 85, S = 255 |
| **L7** | `_MOUSE` | 🟡 **Amarelo** | H = 43, S = 255 |

---

## ⚠️ Limitações Conhecidas do RGB

* **Sobrescrita de Efeitos:** Como o callback `rgb_matrix_indicators_user()` aplica uma cor sólida em todo o buffer a cada ciclo de atualização, efeitos visuais dinâmicos (ondas, arco-íris, respiração) e ajustes de matiz/saturação configurados na aba de iluminação do Vial ficam **visualmente sobrepostos** pela cor da camada ativa.
* **Cobertura de Camadas (L0–L7 vs L8–L15):** A paleta de cores mapeia explicitamente as camadas L0 a L7. Como o Vial suporta até 16 camadas, quaisquer camadas de L8 a L15 cairão no caso padrão (`default`), acendendo em Branco.
* **Escopo de Hardware (Underglow vs Per-Key):** A chamada `rgb_matrix_set_color_all()` escreve em todos os índices da matriz lógica WS2812. No Tergo Sofle com apenas 12 LEDs de underglow, funciona perfeitamente; em eventuais unidades que possuam LEDs individuais por tecla (*per-key RGB*), os LEDs dos switches também assumirão a cor da camada ativa.
* **Liga / Desliga:** A tecla `RGB_TOG` (na Camada 4) continua permitindo desligar os LEDs por completo (o QMK suspende a renderização de indicadores quando o subsistema RGB está desativado).
* **Segurança Elétrica:** A contenção de valor a 110 limita a intensidade no cálculo do software, em conformidade com o comentário do código oficial (`// Não aumentar. Pode diminuir.`), mas não constitui laudo ou certificação elétrica de hardware.


---

## 🛠️ O Código do Patch (33 linhas adicionadas)

Adicionado ao final de `keyboards/tergo_sofle/keymaps/default/keymap.c`:

```c
// Color all LEDs according to the active layer.
// Since this keyboard has only 12 underglow LEDs physically connected,
// setting all LEDs ensures the color reaches the physical LEDs regardless of PCB trace indexing.
bool rgb_matrix_indicators_user(void) {
    uint8_t layer = get_highest_layer(layer_state);
    if (layer == 0) {
        layer = get_highest_layer(default_layer_state);
    }

    uint8_t val = rgb_matrix_get_val();
    if (val > RGB_MATRIX_MAXIMUM_BRIGHTNESS) {
        val = RGB_MATRIX_MAXIMUM_BRIGHTNESS;
    }

    HSV hsv = { .h = 0, .s = 0, .v = val }; // Default white (S=0)

    switch (layer) {
        case _BASIC:         hsv.s = 0;   break; // Branco (L0)
        case _ADEPT:         hsv.h = 24;  hsv.s = 255; break; // Laranja (L1)
        case _CUSTOM:        hsv.h = 128; hsv.s = 255; break; // Ciano (L2)
        case _SYMB_AND_NAV:  hsv.h = 170; hsv.s = 255; break; // Azul (L3)
        case _ADJUST:        hsv.h = 0;   hsv.s = 255; break; // Vermelho (L4)
        case _MEDIA:         hsv.h = 195; hsv.s = 255; break; // Roxo (L5)
        case _NUMPAD:        hsv.h = 85;  hsv.s = 255; break; // Verde (L6)
        case _MOUSE:         hsv.h = 43;  hsv.s = 255; break; // Amarelo (L7)
        default:             hsv.s = 0;   break; // Branco (Outras)
    }

    RGB rgb = hsv_to_rgb(hsv);
    rgb_matrix_set_color_all(rgb.r, rgb.g, rgb.b);
    return false;
}
```

---

## 📥 Procedimento de Instalação e Teste

### 1. Salvaguarda Prévia Obrigatória (Backup do Layout)
1. Abra o aplicativo **Vial** antes de alterar o teclado.
2. Acesse o menu **File > Save current layout...** e salve uma cópia do seu layout atual em um local seguro (ex: `backup_meu_layout.vil`).
   > [!IMPORTANT]
   > **Atenção ao BUILD_ID e Reset de EEPROM:** Toda nova compilação do QMK/Vial gera um identificador aleatório (`BUILD_ID`). Nesta base Tergo/Vial, se o `BUILD_ID` diferir do registrado, o firmware reinicializa o mapa e as macros na primeira inicialização após a gravação (`dynamic_keymap_reset()`). **O backup é fundamental porque seu layout personalizado e macros serão reinicializados durante o processo.**
3. Tenha em mãos o firmware oficial de retorno fornecido pela Tergo Teclados.

### 2. Sequência Segura de Desconexão e Gravação
> [!CAUTION]
> **Risco com Cabo TRRS:** Conectar ou desconectar cabos TRRS com o teclado energizado via USB pode colocar pinos de alimentação em contato direto com pinos de dados GPIO, danificando permanentemente os microcontroladores RP2040. **Nunca mexa no cabo TRRS com nenhum cabo USB conectado.**

Siga rigorosamente esta ordem:
1. **Desconexão total:** Desconecte todos os cabos USB do computador e do teclado.
2. **Desconectar o TRRS:** Com o teclado completamente desenergizado, desconecte o cabo TRRS que une as duas metades.
3. **Gravar a Metade Esquerda:**
   * Conecte apenas a metade esquerda ao computador via USB.
   * Coloque-a em modo bootloader (duplo clique rápido no botão de reset físico na placa controladora ou atalho de bootloader). O volume `RPI-RP2` será montado.
   * Copie o arquivo `Tergo_Sofle_v108_layer_rgb_v03.uf2` para o volume `RPI-RP2`. O teclado reiniciará automaticamente.
   * **Desconecte o cabo USB da metade esquerda.**
4. **Gravar a Metade Direita:**
   * Conecte apenas a metade direita ao computador via USB.
   * Coloque-a em modo bootloader (duplo clique rápido no botão de reset físico).
   * Copie o mesmo arquivo `Tergo_Sofle_v108_layer_rgb_v03.uf2` para o volume `RPI-RP2`.
   * **Desconecte o cabo USB da metade direita.**
5. **Reconexão segura:**
   * Com **todos** os cabos USB desconectados, conecte firmemente o cabo TRRS entre as duas metades.
   * **Somente após o TRRS estar conectado e firme**, plugue o cabo USB principal na metade controladora (metade esquerda).

### 3. Restauração do Layout e Validação
1. **Restauração no Vial:**
   * Abra o aplicativo **Vial**. Como o `BUILD_ID` difere do firmware anterior, o layout terá voltado ao padrão de fábrica da Tergo.
   * Acesse **File > Load saved layout...** e selecione o seu arquivo de backup (`backup_meu_layout.vil`) para restaurar seu mapa de teclas e macros.
2. **Validação das Cores:**
   * O underglow acenderá na cor da camada base ativa no momento (Branco para L0 `_BASIC`, Laranja para L1 `_ADEPT`, ou Ciano se sua base for L2 `_CUSTOM`).
   * Ao segurar a tecla de símbolos (L3), a iluminação deve transitar para **Azul**; ao soltar, deve retornar imediatamente à cor da sua camada base.
   * Ao alternar para o teclado numérico (L6) ou modo ajuste (L4), confirme as cores correspondentes (Verde e Vermelho, respectivamente).
   * Teste a tecla `RGB_TOG` (Camada 4) para verificar se o comando de ligar/desligar a iluminação continua funcional.

---

## 🔄 Como Retornar ao Firmware de Fábrica (Rollback)

Caso queira reverter ao firmware original a qualquer momento, repita a mesma sequência segura:
1. **Desconectar:** Desconecte o cabo USB principal e, com o teclado desenergizado, desconecte o cabo TRRS entre as metades.
2. **Restaurar Metade Esquerda:** Conecte apenas a metade esquerda via USB, acione o bootloader (`RPI-RP2`), copie o arquivo `.uf2` oficial original da Tergo e desconecte o cabo USB.
3. **Restaurar Metade Direita:** Conecte apenas a metade direita via USB, acione o bootloader, copie o mesmo `.uf2` oficial original da Tergo e desconecte o cabo USB.
4. **Reconectar com Segurança:** Com todos os cabos USB desligados, conecte firmemente o cabo TRRS. Em seguida, conecte o cabo USB principal na metade esquerda.
5. **Restaurar Layout:** Abra o Vial e recarregue o backup salvo via **File > Load saved layout...**.

---

## 📦 Como Compilar a Partir do Repositório Oficial

Para auditar o código e gerar um binário independente:

```bash
# 1. Clonar o repositório oficial da Tergo Teclados
git clone https://github.com/TergoTeclados/vial-qmk-firmware.git
cd vial-qmk-firmware
git checkout tergo-sofle-v1.08
git submodule update --init --recursive

# 2. Aplicar o patch limpo
git apply /caminho/para/tergo-sofle-v108-layer-rgb.patch

# 3. Compilar para RP2040 (versão com fio oficial)
qmk compile -kb tergo_sofle/rev1 -km default
```

O arquivo binário gerado estará em `.build/tergo_sofle_rev1_default_rp2040_ce.uf2`.

---

## ⚡ Binário de Referência e Rastreabilidade

O arquivo binário pré-compilado na pasta `release/` é disponibilizado como referência de teste da sessão de validação física de 20/10/2026:
* **Arquivo:** `release/Tergo_Sofle_v108_layer_rgb_v03.uf2`
* **Target:** `tergo_sofle/rev1:default CONVERT_TO=rp2040_ce`
* **Checksum SHA-256:** `a8df043e2c4c275d0d37d6c4df454b521fd8871fab91cd340c181800d7397413`

Para conferir a integridade do arquivo no terminal:
```bash
# A partir da raiz do pacote:
shasum -a 256 -c SHA256SUMS.txt

# Ou a partir da pasta release/:
cd release && shasum -a 256 -c SHA256SUMS.txt
```

> [!NOTE]
> **Rastreabilidade e Correspondência de Compilação:**  
> A verificação independente realizada sobre a árvore oficial `tergo-sofle-v1.08` com o patch aplicado confirmou a correspondência do código com o binário: o `.uf2` recém-compilado possui exatamente o mesmo tamanho e estrutura do binário de referência distribuído, diferindo em apenas 8 bytes. Esses 8 bytes correspondem estritamente ao identificador aleatório `BUILD_ID` gerado a cada compilação pelo script `util/build_id.py` do QMK/Vial (utilizado na validação da compatibilidade dos dados salvos). O restante do payload é idêntico; as diferenças estão restritas às constantes do `BUILD_ID`. A lógica de indicadores RGB, matriz e camadas permanecem rigorosamente idênticas.
> 
> Quem desejar compilar seu próprio binário diretamente a partir do código-fonte para auditoria completa e conformidade com a GPLv2 pode seguir o procedimento acima.

---

## 📜 Licença e Código-Fonte

* **Projeto Base:** [vial-qmk-firmware](https://github.com/TergoTeclados/vial-qmk-firmware) da Tergo Teclados.
* **Copyright:** Copyright (C) 2024 Tergo Teclados.
* **Licença:** Distribuído sob os termos da **GNU General Public License v2 (GPL-2.0-or-later)**, conforme autorizado no cabeçalho do código oficial da Tergo (`version 2 of the License, or (at your option) any later version`).
* **Código de Modificação:** A alteração proposta está documentada integralmente no arquivo [tergo-sofle-v108-layer-rgb.patch](tergo-sofle-v108-layer-rgb.patch), aplicável sobre a tag oficial `tergo-sofle-v1.08` (commit `2afaaf575d94da2b02b4b50c6deca11d542616c5`). Consulte o arquivo [LICENSE](LICENSE) para o texto integral da licença.

