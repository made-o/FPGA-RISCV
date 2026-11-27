# MEM → Memory Access

## Resumen

Módulo que:
- tiene acceso a memoria de datos:
  - lee (loads) o 
  - escribe (stores) *según las señales de control que vienen de EX*

## Archivos

- `MEM/memory.v` → instancia la memoria de datos y contiene el latch MEM/WB. (El latch EX/MEM está en `EX/IExecute.v`.)
- `MEM/dataMemory.v` → la memoria de datos en sí.

## Diagrama

```mermaid
flowchart LR
    subgraph MEMS["Etapa MEM (memory.v)"]
        DM[("dataMemory.v<br/>128 palabras<br/>lectura combinacional ·<br/>escritura en el flanco")]
        LATCH[["Latch MEM/WB"]]
    end

    EXMEM["Latch EX/MEM<br/>(en IExecute.v)"] -->|"aluResult[8:0]<br/>dirección en bytes"| DM
    EXMEM -->|"datoLeido2<br/>dato del store"| DM
    EXMEM -->|"memRead · memWrite ·<br/>mem_size · mem_unsigned"| DM

    DM -->|"dato leído<br/>(byte/half/word, con o sin signo)"| LATCH
    EXMEM -->|"aluResult · rd ·<br/>memToReg · regWrite · halt"| LATCH
    LATCH --> WB["Etapa WB"]

    EXMEM -.->|"aluResult (forwarding)"| FW["EX y hazard_detector (ID)"]

    DBG["Debug Manager"] -.->|"i_addr_tx_MEM<br/>(índice de palabra)"| DM
    DM -.->|"o_data_send_tx_MEM<br/>(puerto registrado)"| DBG

    style DM fill:#0e7c86,color:#fff,stroke:#0e7c86
```

> *El puerto del Debug Manager es independiente del acceso del programa. Usa un índice de palabra directo de 7 bits, por eso puede leer las 128 palabras.*


## Señales clave

| Señal | Sentido | Qué es |
|---|---|---|
| `i_memRead`, `i_memWrite` | entrada | habilitan lectura/escritura |
| `i_aluResult` | entrada | dirección de memoria (viene de la ALU, registrada en EX/MEM) |
| `i_datoLeido2` | entrada | dato a escribir en un store (el `rs2` ya adelantado) |
| `i_mem_size`, `i_mem_unsigned` | entrada | tamaño del acceso (byte/half/word) y signo → para LB/LH/LBU/LHU |
| `o_readData`, `o_aluResult`, `o_rd` | salida | van al latch MEM/WB |

## ✨ Particularidades

- El tamaño y signo del acceso (`i_mem_size`/`i_mem_unsigned`) se extraen directamente de la instrucción en EX (`i_instruccion[13:12]` y `i_instruccion[14]`, campos del `func3` de loads/stores en RISC-V), no de una señal de control dedicada. Se guardan en el latch EX/MEM y llegan acá.

- La memoria es **little-endian** y se accede por byte, 
  - media palabra o palabra: `sb`/`sh` modifican solo la parte correspondiente de la palabra, 
  - y `lb`/`lh` extienden signo (o completan con ceros en `lbu`/`lhu`).

- **La lectura es combinacional** (desde el latch EX/MEM) y el resultado se registra en MEM/WB; **la escritura** ocurre en el flanco de clock.

- **Las 128 palabras son accesibles.** `dataMemory` declara 128 palabras (`RAM_DEPTH = 128`) y `memory.v` le pasa los 9 bits bajos de la dirección en bytes (`i_aluResult[8:0]`, o sea `[$clog2(RAM_DEPTH)+1:0]`). La memoria indexa la palabra con `[8:2]` (`w_word_addr`): **son 512 bytes = 128 palabras**, bytes 0 a 511.
  - Una dirección ≥ 512 se enrosca módulo 512 (por ejemplo, un store al byte 512 pisa la palabra 0).
  - El volcado por UART recorre las mismas 128 palabras, así que todo lo que aparece en el volcado lo puede modificar el programa.
  - `memory/memData.mem` inicializa solo las primeras 32 palabras. En simulación el resto arranca en `x`.

- Los valores iniciales se cargan con `$readmemb` desde `memory/memData.mem` (ruta relativa en `FILE_DATA`. El **reset no los restaura**: después de un reset **la memoria conserva lo que dejó la corrida anterior**.

- No hay hazard estructural que resolver acá → MEM no necesita saber nada de otras etapas más que lo que le llega por el pipeline.
