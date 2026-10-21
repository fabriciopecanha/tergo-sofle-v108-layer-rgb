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
* **Liga / Desliga:** A tecla `RGB_TOG` (na Camada 4) continua permitindo desligar os LEDs por completo (o QMK suspende a renderização de indicadores quando o subsistema RGB está desativado).
* **Escopo de Hardware:** A chamada `rgb_matrix_set_color_all()` escreve em todos os índices da matriz lógica, garantindo que o sinal alcance os LEDs físicos sem depender do mapeamento de flags de underglow. A topologia exata das trilhas físicas cabe à fabricante confirmar.
* **Segurança Elétrica:** A contenção de valor a 110 limita a intensidade no cálculo do software, em conformidade com o comentário do código oficial (`// Não aumentar. Pode diminuir.`), mas não constitui laudo ou certificação elétrica de hardware.

---

## 🛠️ O Código do Patch (32 linhas)

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

### 1. Salvaguarda Prévia Obrigatória
1. Abra o aplicativo **Vial** antes de alterar o teclado.
2. Acesse o menu **File > Save current layout...** e salve uma cópia do seu layout atual em um local seguro (ex: `backup_meu_layout.vil`).
3. Tenha em mãos o firmware oficial de retorno fornecido pela Tergo Teclados.

### 2. Gravação do Firmware
1. Conecte a **metade esquerda** via cabo USB ao computador.
2. Coloque a metade em modo bootloader (duplo clique rápido no botão de reset físico na placa controladora ou atalho de bootloader do teclado). O volume `RPI-RP2` será montado no computador.
3. Copie o arquivo `Tergo_Sofle_v108_layer_rgb_v03.uf2` para o volume `RPI-RP2`. O teclado reiniciará automaticamente.
4. Desconecte e repita exatamente o mesmo procedimento de gravação na **metade direita**.
5. **Reconexão segura:** Com todos os cabos USB desconectados da energia, conecte firmemente o cabo TRRS entre as duas metades. **Somente após o TRRS estar conectado**, plugue o cabo USB principal na metade controladora (conectar ou desconectar cabos TRRS com o teclado energizado pode causar curto-circuito nos pinos dos controladores).

### 3. Validação
* O underglow deve acender na cor da sua camada base ativa no momento (Branco para L0 `_BASIC`, Laranja para L1 `_ADEPT`, ou Ciano para L2 `_CUSTOM`).
* Ao segurar a tecla de símbolos (L3), a iluminação deve transitar para **Azul**; ao soltar, deve retornar à cor da camada base ativa (ex: Branco ou Ciano).
* Abra o Vial e verifique se as teclas continuam respondendo e customizáveis normalmente.

---

## 🔄 Como Retornar ao Firmware de Fábrica (Rollback)

Caso queira reverter a qualquer momento:
1. Coloque a metade esquerda em modo bootloader (montando `RPI-RP2`).
2. Copie o arquivo `.uf2` oficial original da Tergo para a unidade.
3. Repita o processo na metade direita com o firmware oficial.
4. Abra o Vial e restaure o seu layout salvo via **File > Load saved layout...**.

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

> [!IMPORTANT]
> **Nota sobre Rastreabilidade:** O checksum confirma unicamente a integridade do arquivo `.uf2` distribuído. Uma rotina de compilação automatizada que comprove a ligação formal e reprodutível entre este binário e o código do patch não foi executada nesta auditoria. Para garantir correspondência exata e atender aos requisitos de compilação da GPLv2, recomenda-se que a avaliação técnica e o uso sejam feitos compilando diretamente a partir do `.patch`.

---

## 📜 Licença e Código-Fonte

* **Projeto Base:** [vial-qmk-firmware](https://github.com/TergoTeclados/vial-qmk-firmware) da Tergo Teclados.
* **Copyright:** Copyright (C) 2024 Tergo Teclados.
* **Licença:** Distribuído sob os termos da **GNU General Public License v2 (GPL-2.0-or-later)**, conforme autorizado no cabeçalho do código oficial da Tergo (`version 2 of the License, or (at your option) any later version`).
* **Código de Modificação:** A alteração proposta está documentada integralmente no arquivo [tergo-sofle-v108-layer-rgb.patch](tergo-sofle-v108-layer-rgb.patch), aplicável sobre a tag oficial `tergo-sofle-v1.08` (commit `2afaaf575d94da2b02b4b50c6deca11d542616c5`). Consulte o arquivo [LICENSE](LICENSE) para o texto integral da licença.

