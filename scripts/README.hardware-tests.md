# Hardware Compatibility Testing Docker Environment

Este ambiente Docker fornece todas as ferramentas necessárias para executar os testes de compatibilidade de hardware descritos em `research/guides/hardware-compatibility-check.md`, sem precisar instalar pacotes diretamente na máquina host.

## Construindo a Imagem

Para construir a imagem Docker para ARM64:

```bash
docker build -f scripts/Dockerfile.hardware-tests -t hardware-tests:arm64 .
```

## Executando os Testes

### Modo Interativo (Recomendado)

Para acesso completo ao hardware do host, execute com privilégios e acesso aos dispositivos:

```bash
docker run --rm -it \
  --privileged \
  --network host \
  -v /dev:/dev \
  -v /sys:/sys:ro \
  -v /proc:/proc:ro \
  -v $(pwd)/results:/tests/results \
  hardware-tests:arm64
```

Dentro do container, você pode executar:

```bash
# Verificação básica de hardware
/tests/hardware-check.sh

# Teste de stress da CPU (30 minutos por padrão, ou especifique segundos)
/tests/cpu-stress-test.sh 300

# Teste de memória (tamanho em MB e número de iterações)
/tests/memory-test.sh 512 5

# Teste de armazenamento (especifique o dispositivo)
/tests/storage-test.sh /dev/sda

# Teste de rede
/tests/network-test.sh

# Executar todos os testes de uma vez
/tests/run-all-tests.sh /dev/sda
```

### Modo Não-Privilegiado (Limitado)

Se não quiser executar com `--privileged`, você ainda pode rodar alguns testes básicos:

```bash
docker run --rm -it \
  --network host \
  -v $(pwd)/results:/tests/results \
  hardware-tests:arm64
```

**Nota:** Neste modo, alguns testes que requerem acesso direto ao hardware (dmidecode, smartctl, ethtool) não funcionarão completamente.

## Scripts Disponíveis

### `/tests/hardware-check.sh`
Coleta informações abrangentes sobre o hardware:
- CPU (modelo, cores, arquitetura)
- Memória (capacidade, velocidade, tipo)
- Armazenamento (dispositivos, espaço)
- Rede (interfaces, velocidades)
- Temperatura e sensores

### `/tests/cpu-stress-test.sh [duração_em_segundos]`
Testa estabilidade da CPU sob carga máxima:
- Uso padrão: 30 minutos
- Monitora temperaturas durante o teste
- Exemplo: `./cpu-stress-test.sh 600` (10 minutos)

### `/tests/memory-test.sh [tamanho_mb] [iterações]`
Testa integridade da memória RAM:
- Padrão: 1024MB, 5 iterações
- Exemplo: `./memory-test.sh 2048 10` (2GB, 10 iterações)

### `/tests/storage-test.sh [dispositivo]`
Verifica saúde e desempenho do armazenamento:
- Padrão: `/dev/sda`
- Requer acesso root para SMART e testes de velocidade
- Exemplo: `./storage-test.sh /dev/nvme0n1`

### `/tests/network-test.sh`
Testa interfaces de rede:
- Lista todas as interfaces
- Verifica status de link e velocidades
- Sugere uso de iperf3 para testes de throughput

### `/tests/run-all-tests.sh [dispositivo_storage]`
Executa todos os testes em sequência:
- Salva resultados em `/tests/results/` com timestamp
- Exemplo: `./run-all-tests.sh /dev/sda`

## Montando Volumes para Resultados

Os resultados dos testes são salvos em `/tests/results/` dentro do container. Para persistir os resultados:

```bash
mkdir -p results
docker run --rm -it \
  --privileged \
  --network host \
  -v /dev:/dev \
  -v /sys:/sys:ro \
  -v /proc:/proc:ro \
  -v $(pwd)/results:/tests/results \
  hardware-tests:arm64 \
  /tests/run-all-tests.sh /dev/sda
```

Os arquivos de resultado terão nomes como:
- `hardware-check_20250108_220000.txt`
- `cpu-stress_20250108_220500.txt`
- `memory-test_20250108_221000.txt`
- etc.

## Testes de Rede com iperf3

Para testar throughput de rede entre dois sistemas:

**No servidor:**
```bash
docker run --rm -it \
  --network host \
  hardware-tests:arm64 \
  iperf3 -s
```

**No cliente:**
```bash
docker run --rm -it \
  --network host \
  hardware-tests:arm64 \
  iperf3 -c <ip_do_servidor>
```

## Ferramentas Incluídas

A imagem inclui:
- **Detecção de hardware:** lshw, dmidecode, lspci, lsusb
- **Monitoramento:** lm-sensors, htop, sysstat
- **CPU:** stress, stress-ng, cpuid
- **Memória:** memtester
- **Armazenamento:** smartmontools, hdparm, fio
- **Rede:** ethtool, iperf3, net-tools, iproute2

## Notas Importantes

1. **Acesso Privilegiado:** Muitas ferramentas de diagnóstico de hardware requerem acesso privilegiado (root) e acesso direto aos dispositivos do sistema. Use `--privileged` para funcionalidade completa.

2. **Segurança:** Execute containers privilegiados apenas em ambientes controlados e confiáveis.

3. **ARM64:** Esta imagem foi construída especificamente para arquitetura ARM64. Para outras arquiteturas, ajuste a imagem base conforme necessário.

4. **Burn-in Testing:** Para testes de estabilidade de 24-48 horas mencionados no guia, ajuste os parâmetros de duração dos scripts ou execute-os em loop.

## Exemplo de Workflow Completo

```bash
# 1. Construir a imagem
docker build -f scripts/Dockerfile.hardware-tests -t hardware-tests:arm64 .

# 2. Criar diretório para resultados
mkdir -p results

# 3. Executar todos os testes
docker run --rm \
  --privileged \
  --network host \
  -v /dev:/dev \
  -v /sys:/sys:ro \
  -v /proc:/proc:ro \
  -v $(pwd)/results:/tests/results \
  hardware-tests:arm64 \
  /tests/run-all-tests.sh /dev/sda

# 4. Revisar resultados
ls -lh results/
cat results/hardware-check_*.txt
```

## Referência

Este ambiente Docker implementa os testes descritos em:
- `research/guides/hardware-compatibility-check.md`

Para detalhes completos sobre procedimentos de teste e interpretação de resultados, consulte o guia original.
