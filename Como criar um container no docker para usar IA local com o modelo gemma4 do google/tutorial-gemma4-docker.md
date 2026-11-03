# Instalar Gemma 4 E4B no Docker com llama.cpp (CUDA)

Tutorial para baixar e executar o modelo **Gemma 4 E4B Uncensored
HauhauCS Aggressive Q4_K_M**, com suporte a imagens por meio do arquivo
`mmproj`, usando Docker Desktop e uma GPU NVIDIA no Windows 11.

> **Ambiente considerado:** Windows 11, PowerShell, Docker Desktop com
> backend WSL2 e GPU NVIDIA.
>
> **Importante:** os nomes dos arquivos, o repositório Hugging Face e os
> comandos abaixo correspondem à variante de modelo usada neste
> tutorial. Confira se os arquivos continuam disponíveis no repositório
> antes de iniciar o download.

## Índice

1.  [Pré-requisitos e diagnóstico](#1-pré-requisitos-e-diagnóstico)
2.  [Baixar os arquivos do modelo](#2-baixar-os-arquivos-do-modelo)
3.  [Ajustar a configuração conforme o
    hardware](#3-ajustar-a-configuração-conforme-o-hardware)
4.  [Baixar a imagem Docker](#4-baixar-a-imagem-docker)
5.  [Executar o container](#5-executar-o-container)
6.  [Acompanhar a inicialização](#6-acompanhar-a-inicialização)
7.  [Testar a API](#7-testar-a-api)
8.  [Comandos de manutenção](#9-comandos-de-manutenção)
9. [Solução de problemas](#10-solução-de-problemas)

------------------------------------------------------------------------

## 1. Pré-requisitos e diagnóstico

Antes de começar, verifique se você possui:

-   Docker Desktop instalado e iniciado, preferencialmente com backend
    WSL2.
-   Driver NVIDIA atualizado.
-   GPU NVIDIA compatível com CUDA.
-   Espaço em disco suficiente para os arquivos do modelo e para a
    imagem Docker.
-   PowerShell.

### 1.1. Verificar o driver e a GPU

Abra o PowerShell e execute:

``` powershell
nvidia-smi
```

O comando deve exibir o modelo da GPU, a versão do driver e a memória de
vídeo (VRAM).

### 1.2. Verificar o Docker

``` powershell
docker --version
docker info
```

### 1.3. Testar o acesso à GPU pelo Docker

Execute:

``` powershell
docker run --rm --gpus all `
  nvidia/cuda:12.8.1-base-ubuntu24.04 `
  nvidia-smi
```

Se o comando exibir a GPU dentro do container, o Docker está conseguindo
disponibilizá-la.

Se houver erro, verifique se o Docker Desktop está iniciado, se o
backend WSL2 está habilitado e se o driver NVIDIA está atualizado. Não é
necessário instalar o CUDA Toolkit completo no Windows apenas para
executar esse container; o driver precisa ser compatível com a versão
CUDA usada pela imagem.

------------------------------------------------------------------------

## 2. Baixar os arquivos do modelo

O modelo precisa de dois arquivos:

  --------------------------------------------------------------------------------------------------
  Arquivo                                                        Finalidade
  -------------------------------------------------------------- -----------------------------------
  `Gemma-4-E4B-Uncensored-HauhauCS-Aggressive-Q4_K_M.gguf`       Pesos do modelo principal

  `mmproj-Gemma-4-E4B-Uncensored-HauhauCS-Aggressive-f16.gguf`   Projetor multimodal, utilizado para
                                                                 processar imagens
  --------------------------------------------------------------------------------------------------

Repositório Hugging Face:

-   https://huggingface.co/HauhauCS/Gemma-4-E4B-Uncensored-HauhauCS-Aggressive

### 2.1. Definir o diretório dos modelos

O exemplo utiliza `D:\AI\Models\Gemma4`. Altere o caminho se preferir
armazenar os modelos em outra unidade.

``` powershell
$ModelDir = "D:\AI\Models\Gemma4"

New-Item -ItemType Directory -Force -Path $ModelDir
```

### 2.2. Instalar a ferramenta de download

``` powershell
py -m pip install -U huggingface_hub
```

Se o comando `hf` não for reconhecido após a instalação, feche e abra
novamente o PowerShell. Também é possível executar
`py -m pip show huggingface_hub` para verificar a instalação.

### 2.3. Baixar o modelo principal

``` powershell
hf download `
  HauhauCS/Gemma-4-E4B-Uncensored-HauhauCS-Aggressive `
  Gemma-4-E4B-Uncensored-HauhauCS-Aggressive-Q4_K_M.gguf `
  --local-dir $ModelDir
```

### 2.4. Baixar o projetor multimodal

``` powershell
hf download `
  HauhauCS/Gemma-4-E4B-Uncensored-HauhauCS-Aggressive `
  mmproj-Gemma-4-E4B-Uncensored-HauhauCS-Aggressive-f16.gguf `
  --local-dir $ModelDir
```

### 2.5. Conferir os arquivos

``` powershell
Get-ChildItem $ModelDir
```

Confirme que os dois arquivos `.gguf` estão presentes. O download pode
ocupar vários gigabytes.

> **Atenção:** não renomeie os arquivos sem atualizar os caminhos
> correspondentes no comando `docker run`.

------------------------------------------------------------------------

## 3. Ajustar a configuração conforme o hardware

O consumo de memória depende da GPU, da VRAM disponível, do contexto, do
cache KV, do processamento multimodal e de outros programas em execução.

O script abaixo coleta informações básicas e sugere um contexto inicial.
A sugestão é apenas um ponto de partida: ela não garante que o modelo
caberá integralmente na VRAM.

### 3.1. Executar o diagnóstico

Copie o bloco completo para o PowerShell:

``` powershell
Write-Host "`n===== DIAGNOSTICO DO HARDWARE =====" -ForegroundColor Cyan

# GPU NVIDIA
$GpuName = nvidia-smi --query-gpu=name --format=csv,noheader
$VramText = nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits

if (-not $VramText) {
    throw "Não foi possível identificar a VRAM. Verifique o driver NVIDIA e o comando nvidia-smi."
}

$Vram = [int]($VramText | Select-Object -First 1)

Write-Host "`nGPU: $GpuName"
Write-Host "VRAM total: $Vram MB"

# RAM instalada
$Ram = Get-CimInstance Win32_ComputerSystem
$RamGB = [math]::Round($Ram.TotalPhysicalMemory / 1GB, 1)

Write-Host "`nRAM instalada: $RamGB GB"

# CPU
$Cpu = Get-CimInstance Win32_Processor
Write-Host "CPU: $($Cpu.Name)"

# Sugestão inicial de contexto
if ($Vram -lt 8192) {
    $Context = 4096
}
elseif ($Vram -lt 12288) {
    $Context = 8192
}
elseif ($Vram -lt 16384) {
    $Context = 12288
}
else {
    $Context = 16384
}

Write-Host "`n===== CONFIGURACAO SUGERIDA =====" -ForegroundColor Green
Write-Host "Contexto inicial: $Context tokens"
```

### 3.2. Referência de contexto inicial

       VRAM total   Contexto inicial sugerido
  --------------- ---------------------------
         Até 8 GB                4.096 tokens
        8 a 12 GB                8.192 tokens
       12 a 16 GB               12.288 tokens
    16 GB ou mais               16.384 tokens

Esses valores são referências conservadoras, não uma regra fixa. A
capacidade real depende da memória livre e das demais configurações.

### 3.3. Verificar a memória disponível

``` powershell
nvidia-smi
```

Observe a coluna de memória utilizada. Se houver outros programas usando
a GPU, feche-os antes de iniciar o modelo, se possível.

**Observação sobre o parâmetro `--fit`:** ele ajuda o llama.cpp a
ajustar parâmetros que não foram definidos explicitamente. Não considere
que ele reduzirá automaticamente um contexto definido por `-c`. Se
ocorrer falta de memória, reduza o contexto e recrie o container.

------------------------------------------------------------------------

## 4. Baixar a imagem Docker

Baixe a imagem do servidor llama.cpp com suporte a CUDA:

``` powershell
docker pull ghcr.io/ggml-org/llama.cpp:server-cuda
```

A imagem contém o executável `llama-server` preparado para execução com
CUDA.

------------------------------------------------------------------------

## 5. Executar o container

O comando a seguir usa variáveis para facilitar a personalização. Ajuste
os valores de acordo com o diagnóstico e com os diretórios da máquina.

### 5.1. Definir os parâmetros

``` powershell
# Diretório onde os arquivos GGUF foram baixados
$ModelDir = "D:\AI\Models\Gemma4"

# Contexto sugerido pelo diagnóstico ou definido manualmente
$Context = 16384

# Porta exposta no Windows
$Port = 8081

# Nome do container
$ContainerName = "gemma4-vision"
```

Se o diagnóstico sugeriu outro contexto, substitua `16384` pelo valor
recomendado.

### 5.2. Criar e iniciar o container

``` powershell
docker run -d `
  --name $ContainerName `
  --restart unless-stopped `
  --gpus all `
  -p "${Port}:8080" `
  -v "${ModelDir}:/models" `
  ghcr.io/ggml-org/llama.cpp:server-cuda `
  -m /models/Gemma-4-E4B-Uncensored-HauhauCS-Aggressive-Q4_K_M.gguf `
  --mmproj /models/mmproj-Gemma-4-E4B-Uncensored-HauhauCS-Aggressive-f16.gguf `
  --jinja `
  -c $Context `
  --fit on `
  --host 0.0.0.0 `
  --port 8080
```

### 5.3. Entender os parâmetros

  -----------------------------------------------------------------------
  Parâmetro                           Descrição
  ----------------------------------- -----------------------------------
  `--name`                            Nome do container

  `--restart unless-stopped`          Reinicia o container
                                      automaticamente, exceto quando
                                      parado manualmente

  `--gpus all`                        Disponibiliza as GPUs NVIDIA para o
                                      container

  `-p 8081:8080`                      Mapeia a porta 8080 do container
                                      para a porta 8081 do Windows

  `-v`                                Compartilha a pasta local dos
                                      modelos com o container

  `-m`                                Caminho do modelo principal

  `--mmproj`                          Caminho do projetor multimodal para
                                      processamento de imagens

  `--jinja`                           Habilita o processamento do
                                      template de conversa Jinja

  `-c`                                Define o tamanho do contexto em
                                      tokens

  `--fit on`                          Habilita o ajuste automático de
                                      parâmetros compatíveis pelo
                                      llama.cpp

  `--host 0.0.0.0`                    Permite que o servidor aceite
                                      conexões pelas interfaces de rede
                                      do container

  `--port 8080`                       Porta em que o servidor escuta
                                      dentro do container
  -----------------------------------------------------------------------

> **Segurança:** o exemplo não configura autenticação na API. Por
> padrão, mantenha o serviço acessível apenas na máquina local e não
> exponha a porta diretamente à internet. Se precisar permitir acesso
> pela rede, configure autenticação e regras de firewall adequadas.

------------------------------------------------------------------------

## 6. Acompanhar a inicialização

### 6.1. Ver os logs

``` powershell
docker logs -f gemma4-vision
```

Aguarde o carregamento dos arquivos e a inicialização do servidor. Para
sair da visualização dos logs, pressione `Ctrl+C`; isso não interrompe o
container.

### 6.2. Verificar o container

``` powershell
docker ps
```

### 6.3. Verificar o uso da GPU

Em outro terminal do PowerShell:

``` powershell
nvidia-smi
```

Confira se o processo está utilizando a GPU e se existe memória
suficiente.

------------------------------------------------------------------------

## 7. Testar a API

O llama.cpp disponibiliza uma API compatível com o formato de chat
completions da OpenAI.

### 7.1. Consultar os modelos disponíveis

Abra no navegador:

http://localhost:8081/v1/models

Ou execute:

``` powershell
Invoke-RestMethod -Uri "http://localhost:8081/v1/models"
```

Use o identificador retornado por esse endpoint no campo `model` das
requisições.

### 7.2. Enviar uma mensagem de teste

``` powershell
$Body = @{
    model = "gemma4"
    messages = @(
        @{
            role = "user"
            content = "Explique brevemente o que você consegue fazer."
        }
    )
    temperature = 1.0
} | ConvertTo-Json -Depth 10

Invoke-RestMethod `
    -Uri "http://localhost:8081/v1/chat/completions" `
    -Method Post `
    -ContentType "application/json" `
    -Body $Body
```

Se o identificador `gemma4` não corresponder ao modelo carregado,
substitua-o pelo valor retornado em `/v1/models`.

### 7.3. Testar o acesso pelo navegador

O endereço base do servidor é:

http://localhost:8081

O endpoint de modelos é:

http://localhost:8081/v1/models

------------------------------------------------------------------------


## 8. Comandos de manutenção

  Operação              Comando
  --------------------- --------------------------------
  Listar containers     `docker ps -a`
  Acompanhar logs       `docker logs -f gemma4-vision`
  Parar                 `docker stop gemma4-vision`
  Iniciar               `docker start gemma4-vision`
  Reiniciar             `docker restart gemma4-vision`
  Remover               `docker rm -f gemma4-vision`
  Ver uso de recursos   `docker stats gemma4-vision`

### 8.1. Alterar o contexto

Para alterar o contexto, por exemplo, de 16.384 para 8.192 tokens:

1.  Pare e remova o container atual.
2.  Altere o valor de `$Context`.
3.  Execute novamente o comando da seção 5.2.

Exemplo:

``` powershell
docker rm -f gemma4-vision

$Context = 8192
```

Em seguida, execute novamente o comando `docker run` da seção 5.2.

Os arquivos do modelo continuam no diretório do Windows, portanto não
precisam ser baixados novamente.

------------------------------------------------------------------------

## 9. Solução de problemas

### Erro: `could not select device driver "" with capabilities: [[gpu]]`

O Docker não conseguiu disponibilizar a GPU.

Verifique:

-   Se o driver NVIDIA está instalado e atualizado.
-   Se o Docker Desktop está iniciado.
-   Se o backend WSL2 está habilitado.
-   Se o teste de GPU da seção 1.3 funciona.

### Erro: arquivo de modelo não encontrado

Confira o diretório:

``` powershell
Get-ChildItem "D:\AI\Models\Gemma4"
```

Verifique se os nomes dos arquivos correspondem exatamente aos caminhos
usados em `-m` e `--mmproj`.

### Erro de falta de VRAM

Sintomas comuns incluem falha durante o carregamento, encerramento do
processo ou mensagens de falta de memória nos logs.

Tente:

1.  Fechar aplicações que estejam usando a GPU.
2.  Reduzir o contexto, por exemplo, de `16384` para `8192`.
3.  Reiniciar o container.
4.  Consultar novamente o consumo com `nvidia-smi`.

### O container inicia, mas a API não responde

Consulte os logs:

``` powershell
docker logs --tail 200 gemma4-vision
```

Confira também:

``` powershell
docker ps -a
Test-NetConnection localhost -Port 8081
```

Aguarde o carregamento completo do modelo antes de enviar requisições.

------------------------------------------------------------------------

## Referências

-   Modelo no Hugging Face:
    https://huggingface.co/HauhauCS/Gemma-4-E4B-Uncensored-HauhauCS-Aggressive
-   Repositório llama.cpp: https://github.com/ggml-org/llama.cpp
-   Documentação do servidor llama.cpp:
    https://github.com/ggml-org/llama.cpp/tree/master/examples/server
-   Docker Desktop: https://docs.docker.com/desktop/
-   NVIDIA Container Toolkit:
    https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/

------------------------------------------------------------------------

## Observação final

Os valores de contexto e o comportamento de alocação na GPU variam
conforme a máquina e a versão do llama.cpp. Comece com um contexto
conservador, valide a inicialização e monitore a VRAM antes de aumentar
os parâmetros.
