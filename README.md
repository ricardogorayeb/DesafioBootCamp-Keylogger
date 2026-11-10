# Análise e Defesa contra Malwares: Ransomware & Keylogger

> Projeto educacional — estudo do funcionamento, **detecção**, mitigação e defesa contra ameaças de ransomware e keylogger.
> Ambiente 100% controlado, foco em segurança defensiva (blue team).

---

## ⚠️ Escopo e ética

Este repositório **não contém malware funcional**. Não há código que cifre arquivos
para resgate nem que capture teclas de usuários. O objetivo é **compreender o
comportamento dessas ameaças para saber detectá-las e preveni-las** — a mesma lógica
usada por antivírus, EDRs e equipes de resposta a incidentes.

Todo o código presente aqui é **defensivo**: serve para *detectar* atividade suspeita,
não para causá-la.

---

## 1. Introdução

Ransomware e keyloggers estão entre as ameaças mais comuns e com maior impacto
financeiro no mundo corporativo. Entender como operam é pré-requisito para projetar
defesas eficazes. Este documento descreve, em nível conceitual, o ciclo de vida de
cada ameaça, os **indicadores de comprometimento (IoCs)** que elas deixam, as técnicas
de detecção e um conjunto de controles de defesa em profundidade.

---

## 2. Ransomware

### 2.1 O que é
Software malicioso que torna dados ou sistemas inacessíveis — geralmente por
criptografia — e exige pagamento (resgate) para restaurar o acesso. Variantes
modernas praticam **dupla extorsão**: além de cifrar, exfiltram os dados e ameaçam
publicá-los.

### 2.2 Ciclo de vida (conceitual)
1. **Vetor de entrada** — phishing com anexo/macro, exploração de serviço exposto
   (RDP, VPN desatualizada), ou credencial vazada.
2. **Execução e persistência** — o código roda e garante que sobreviva a reinícios.
3. **Escalonamento e movimentação lateral** — busca privilégios maiores e outros hosts.
4. **Impacto** — cifra arquivos, apaga *shadow copies* e backups acessíveis, deixa
   a nota de resgate.
5. **Extorsão** — exige pagamento, frequentemente em criptomoeda.

### 2.3 Tipos
- **Crypto-ransomware** — cifra arquivos (ex.: documentos, bancos de dados).
- **Locker** — bloqueia o acesso ao sistema inteiro.
- **RaaS (Ransomware-as-a-Service)** — modelo de "afiliados" que aluga a operação.
- **Wiper disfarçado** — finge pedir resgate, mas destrói os dados sem retorno.

### 2.4 Indicadores de comprometimento (IoCs)
- Pico súbito de **alteração/renomeação em massa** de arquivos.
- Extensões incomuns em lote (ex.: `.locked`, `.encrypted`, aleatórias).
- Arquivos de nota de resgate (`README_RECOVER.txt`, `HOW_TO_DECRYPT.html`) em várias pastas.
- Aumento abrupto da **entropia** dos arquivos (conteúdo cifrado parece aleatório).
- Comandos de exclusão de cópias de sombra (`vssadmin delete shadows`), desativação
  de recuperação do sistema.
- Tráfego de saída anômalo (exfiltração) antes da criptografia.

---

## 3. Keylogger

### 3.1 O que é
Ferramenta que registra as teclas digitadas (e por vezes cliques, área de
transferência e capturas de tela), tipicamente para roubar credenciais e dados
sensíveis. Pode ser **software** ou **hardware** (um conector físico entre teclado e
máquina).

### 3.2 Como atua (conceitual)
- **Baseado em API/hook** — intercepta eventos de teclado do sistema operacional.
- **Kernel/driver** — opera em baixo nível, mais difícil de detectar.
- **Form-grabbing** — captura dados de formulários no navegador antes do envio.
- **Exfiltração** — envia o que coletou por rede (HTTP, e-mail, DNS) ou grava localmente
  para coleta posterior.

### 3.3 Indicadores de comprometimento (IoCs)
- Processo desconhecido com gancho global de teclado.
- Programa em inicialização automática sem origem legítima.
- Conexões de saída periódicas para destinos suspeitos.
- Arquivo de log crescendo de forma anômala em diretório temporário/oculto.
- Dispositivo USB não reconhecido entre teclado e computador (keylogger físico).

---

## 4. Detecção

### 4.1 Abordagem comportamental (a mais eficaz)
Assinaturas estáticas falham contra variantes novas. O que realmente funciona é
detectar **comportamento**: criptografia em massa, criação de notas de resgate,
ganchos de teclado, persistência suspeita, exfiltração.

### 4.2 Ferramentas de blue team
| Objetivo | Ferramentas |
|---|---|
| EDR / HIDS | **Wazuh**, OSSEC, Velociraptor |
| Telemetria de endpoint | **Sysmon** (Windows), auditd (Linux) |
| Regras de detecção | **YARA**, Sigma |
| Monitoramento de rede | Zeek, Suricata, firewall **Sophos** |
| SIEM / correlação | Wazuh + Elastic, Graylog |
| Monitoramento de infraestrutura | **Zabbix** |

### 4.3 Exemplo de regra conceitual (Sigma/Wazuh)
- Alertar quando **> N arquivos** forem modificados por um único processo em **< T segundos**.
- Alertar na criação de arquivos com nomes de nota de resgate em múltiplos diretórios.
- Alertar em `vssadmin delete shadows` / desativação de backups.
- Alertar em processo que registra gancho de teclado global fora de uma allowlist.

---

## 5. Defesa e mitigação (defesa em profundidade)

### 5.1 Prevenção
- **Antivírus / EDR** atualizado, com proteção comportamental e *anti-tamper*.
- **Firewall** (ex.: Sophos) com segmentação de rede e bloqueio de saída por padrão.
- **Patch management** — corrigir serviços expostos (sua rotina de Ansible ajuda aqui).
- **Princípio do menor privilégio** — usuários sem admin local; contas de serviço restritas.
- **Application allowlisting** — só executa o que é aprovado (AppLocker, WDAC, fapolicyd no Linux).
- **MFA** em RDP/VPN/e-mail — fecha o vetor de credencial vazada.
- **Desativar macros** por padrão e filtrar anexos no gateway de e-mail.

### 5.2 Contenção e resiliência
- **Backups 3-2-1-1**: 3 cópias, 2 mídias, 1 fora do site, **1 offline/imutável**.
  Teste a restauração — backup não testado não é backup.
- **Segmentação de rede** para limitar movimentação lateral.
- **Sandboxing / detonação** de anexos e binários antes da entrega.
- **Isolamento rápido** do host (quarentena via EDR) ao detectar atividade.

### 5.3 Fator humano
- **Conscientização** — treino anti-phishing, simulações, canal fácil de reporte.
- Políticas claras de uso e resposta a incidentes documentadas e ensaiadas.

### 5.4 Resposta a incidentes (resumo)
Preparação → Identificação → **Contenção** → Erradicação → Recuperação → Lições aprendidas.
Nunca pagar sem envolver jurídico/autoridades; pagamento não garante recuperação e
financia o crime.

---

## 6. Código defensivo — detector de alteração em massa com arquivos-isca (canary)

Demonstração **defensiva** em Python: monitora uma pasta e dispara um alerta se muitos
arquivos forem modificados em pouco tempo — padrão típico de ransomware — usando também
**arquivos-isca** (*canary files*) que não deveriam mudar nunca. Não cifra nem danifica
nada; apenas observa e avisa.

```python
#!/usr/bin/env python3
"""
ransom_canary.py - Detector defensivo de atividade tipo ransomware.
Monitora um diretório: alerta em alteração em massa e em mudança de arquivos-isca.
Uso: python3 -I ransom_canary.py /caminho/para/monitorar
"""
import sys
import time
import hashlib
from pathlib import Path

# Limiares ajustáveis
JANELA_SEG = 10          # intervalo de amostragem
LIMITE_MUDANCAS = 25     # nº de arquivos alterados na janela que dispara alerta
PREFIXO_ISCA = "_canary_"  # arquivos-isca criados pelo monitor


def hash_arquivo(caminho: Path) -> str | None:
    try:
        return hashlib.sha256(caminho.read_bytes()).hexdigest()
    except (OSError, PermissionError):
        return None


def mapear(dir_base: Path) -> dict[Path, str]:
    estado = {}
    for p in dir_base.rglob("*"):
        if p.is_file():
            h = hash_arquivo(p)
            if h:
                estado[p] = h
    return estado


def criar_iscas(dir_base: Path, n: int = 3) -> list[Path]:
    iscas = []
    for i in range(n):
        isca = dir_base / f"{PREFIXO_ISCA}{i}.txt"
        if not isca.exists():
            isca.write_text("NAO EDITAR - arquivo-isca de deteccao.\n")
        iscas.append(isca)
    return iscas


def alerta(msg: str) -> None:
    print(f"\n🚨 ALERTA: {msg}\n")
    # Em produção: enviar para syslog/Wazuh, e-mail, ou acionar isolamento do host.


def main() -> int:
    if len(sys.argv) != 2:
        print("Uso: python3 -I ransom_canary.py <diretorio>")
        return 1

    base = Path(sys.argv[1]).resolve()
    if not base.is_dir():
        print(f"Diretório inválido: {base}")
        return 1

    iscas = criar_iscas(base)
    anterior = mapear(base)
    print(f"Monitorando {base} ({len(anterior)} arquivos). Ctrl+C para sair.")

    try:
        while True:
            time.sleep(JANELA_SEG)
            atual = mapear(base)

            # 1) Arquivos-isca nunca deveriam mudar nem sumir
            for isca in iscas:
                if isca not in atual or atual.get(isca) != anterior.get(isca):
                    alerta(f"Arquivo-isca modificado/removido: {isca.name} "
                           f"(forte indício de ransomware)")

            # 2) Alteração em massa
            mudados = sum(1 for p, h in atual.items()
                          if p in anterior and anterior[p] != h)
            novos = len(set(atual) - set(anterior))
            sumidos = len(set(anterior) - set(atual))
            total = mudados + novos + sumidos
            if total >= LIMITE_MUDANCAS:
                alerta(f"{total} arquivos alterados em {JANELA_SEG}s "
                       f"(modificados={mudados}, novos={novos}, removidos={sumidos})")

            anterior = atual
    except KeyboardInterrupt:
        print("\nMonitor encerrado.")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

**Como testar com segurança:** aponte para uma pasta descartável com arquivos de teste
e simule atividade *criando/renomeando muitos arquivos de texto* à mão (sem qualquer
código malicioso). O monitor deve alertar ao cruzar o limiar.

---

## 7. Reflexão e aprendizados

- Detecção eficaz é **comportamental**, não só baseada em assinatura.
- **Backups imutáveis e testados** são a última e mais importante linha de defesa contra ransomware.
- O **menor privilégio** e a **segmentação** limitam o estrago quando a prevenção falha.
- Contra keyloggers, **MFA** reduz o valor de credenciais roubadas, e **allowlisting**
  mais telemetria de processos ajudam a flagrar o gancho de teclado.
- O **elo humano** (phishing) continua sendo o vetor nº 1 — conscientização tem ROI altíssimo.

---

## 8. Referências para aprofundar

- MITRE ATT&CK — táticas de *Impact* (T1486 Data Encrypted for Impact) e *Collection* (T1056 Input Capture).
- CISA / #StopRansomware — guias de prevenção e resposta.
- Documentação do Wazuh — regras de detecção de ransomware e FIM (File Integrity Monitoring).
- NIST SP 800-61 — Computer Security Incident Handling Guide.
- Projeto YARA e repositórios de regras Sigma.

---

*Projeto desenvolvido para fins educacionais em ambiente controlado, com foco em segurança defensiva.*