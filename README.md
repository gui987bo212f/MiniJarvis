# Jarvis Desk

Assistente de mesa J.A.R.V.I.S. feito com uma **XIAO ESP32-S3 Sense**, um display redondo GC9A01A e um alto-falante. O ESP32 faz o papel de rosto, ouvido, olho e voz; o "cérebro" roda no PC e usa a API do Claude.

- Display com HUD animado; após **30 s sem interação** vira um **relógio de ponteiros animado**.
- Microfone (PDM), câmera e alto-falante (MAX98357A) na mesma placa/USB.
- Segurança: o ESP32 **nunca pede nada ao PC**. Tudo parte do PC por USB serial, e abrir apps ou pastas exige digitar `OPEN` no terminal.

> **Status:** o código foi testado no PC (firmware compilado contra stubs, conversando com o código Python), mas **ainda não foi testado em hardware real**. Veja `docs/hardware.md`, seção "O que foi e o que não foi testado".

## Estrutura

```
firmware/   PlatformIO (Arduino) para a XIAO ESP32-S3 Sense
  include/config.h   pinos
  src/main.cpp       HUD, relógio, microfone, câmera, alto-falante, protocolo serial
pc/         lado do PC (Python)
  jarvis_console.py  console manual (status, record, photo, say, open, sync...)
  jarvis_ai.py       voz -> Claude (com ferramentas) -> fala
docs/hardware.md     peças, ligações, PCB, o que foi testado
test/       testes no PC, sem hardware (./test/run_tests.sh)
```

## Hardware

Peças e ligações completas em [`docs/hardware.md`](docs/hardware.md). Resumo dos pinos (XIAO ESP32-S3 Sense):

| Função | Pino |
|---|---|
| Display SCK / MOSI / CS / DC / RST | D8 / D10 / D0 / D1 / D3 |
| Amplificador DIN / BCLK / LRC | D2 / D4 / D5 |
| Câmera e microfone | internos da Sense |

Ligue o **BLK** do display em 3V3 e **não use cartão microSD** (divide D8/D10 com o display).

## Firmware

```
cd firmware
pio run -t upload
```

Requer [PlatformIO](https://platformio.org/). O `platformio.ini` fixa `espressif32@6.9.0` (Arduino-ESP32 2.0.x).

## PC

```
pip install -r pc/requirements.txt
cd pc

# console manual
python jarvis_console.py --list
python jarvis_console.py COM5

# IA por voz (ENTER = falar, ou digite o texto)
export ANTHROPIC_API_KEY=...        # Windows: set ANTHROPIC_API_KEY=...
export JARVIS_NTFY_TOPIC=um-nome-secreto   # opcional: notificações no celular via ntfy
python jarvis_ai.py COM5
```

Linux precisa de `espeak-ng` para a voz; no Windows e macOS é usada a voz do sistema.

### Comandos do console

`status`, `sleep`, `wake`, `sync` (acerta o relógio), `record` (salva um WAV de 2 s), `photo` (salva um JPEG), `say <texto>`, `open`, `quit`.

### Protocolo serial (USB, 115200)

Uma linha por comando, o PC sempre inicia:

| PC envia | Jarvis responde |
|---|---|
| `STATUS` `SLEEP` `WAKE` `OPENED` | `JARVIS state=HAPPY\|ASLEEP\|TALKING\|RESTING` |
| `TIME <segundos desde 00:00 local>` | `JARVIS state=...` |
| `TALK` `QUIET` | nada |
| `RECORD` | `JARVIS AUDIO 64000` + PCM 16 kHz mono 16-bit |
| `SNAP` | `JARVIS IMAGE <n>` + JPEG |
| `PLAY <bytes>` | `JARVIS READY`, depois um `JARVIS ACK` a cada 2048 bytes enviados |

Erros: `JARVIS ERR <código>`. O texto recebido do ESP32 é só texto de exibição, nunca instrução para o PC.

## Testes

```
./test/run_tests.sh
```

Precisa de `g++`, `python3` e `pillow`. Compila o `main.cpp` real para o PC e o executa contra o código Python real. O CI (`.github/workflows/ci.yml`) roda isso e também `pio run`, que é o que valida a compilação com o toolchain do ESP32.

## Licença

[MIT](LICENSE). Troque o nome em `LICENSE` pelo seu antes de publicar.
