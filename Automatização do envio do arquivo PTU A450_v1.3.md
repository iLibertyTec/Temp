# 📋 Documentação de Processo: Automatização do envio do arquivo PTU A450

### 🎯 Visão Geral do Processo
Este robô automatiza o processo diário de geração, extração e envio do arquivo **PTU A450** (Serviços dos Prestadores) para a Unimed. O arquivo A450 é parte fundamental do **Protocolo de Transação Unimed (PTU)**, regulado pela ANS, e contém informações detalhadas sobre os serviços contratados e oferecidos pelos prestadores da rede.

Assim como no processo do A400, o robô interage com os sistemas internos (SGU, GoGlobal) para garantir a integridade e pontualidade na troca de informações cadastrais e de serviços.

### 🎯 Objetivo e Escopo
- **O que automatiza**: 
  1. Geração do arquivo A450 no sistema SGU.
  2. Download e compactação do arquivo gerado.
  3. Envio do arquivo processado por e-mail.
  4. Organização e backup dos arquivos.
- **Benefícios**: Assegura que a atualização dos serviços dos prestadores seja comunicada diariamente sem intervenção manual, reduzindo o risco de inconsistências na rede credenciada.
- **Frequência**: Execução diária.

## 🔄 Como o Processo Funciona

O diagrama abaixo ilustra o fluxo de execução do robô:

```mermaid
graph TD
    Start([Início]) --> Params[Configuração de Parâmetros]
    Params --> DB[Verificação de Banco de Dados e Checkpoints]
    DB --> DateCheck{Novo Dia?}
    DateCheck -- Sim --> Reset[Resetar Checkpoints Diários]
    DateCheck -- Não --> TryStart[Início do Fluxo Principal]
    Reset --> TryStart
    
    subgraph "Processamento A450 (SGU/GIU)"
        TryStart --> GenCheck{Geração Concluída?}
        GenCheck -- Não --> LoginSGU[Login no SGU via GoGlobal]
        LoginSGU --> GenAction[Gerar Arquivo A450]
        GenAction --> SaveGen[Salvar Checkpoint: Gerado]
        SaveGen --> GetCheck
        GenCheck -- Sim --> GetCheck{Download Concluído?}
        
        GetCheck -- Não --> LoginREP[Acessar SGU REP]
        LoginREP --> Download[Baixar Arquivo A450]
        Download --> Zip[Compactar Arquivos (.zip)]
        Zip --> SaveGet[Salvar Checkpoint: Baixado]
        SaveGet --> SendCheck
        GetCheck -- Sim --> SendCheck{Envio Concluído?}
        
        SendCheck -- Não --> SendAction[Enviar Email com A450]
        SendCheck -- Sim --> Backup
    end
    
    TryStart -. Erro .-> ErrorHandler[Registrar Erro no Banco de Dados]
    
    SendAction --> Backup[Mover Arquivos para Pasta de Backup]
    Backup --> FinalEmail[Enviar Email de Confirmação]
    FinalEmail --> End([Fim])
    
    style Start fill:#4CAF50,stroke:#2E7D32,color:#fff
    style End fill:#4CAF50,stroke:#2E7D32,color:#fff
    style ErrorHandler fill:#f44336,stroke:#b71c1c,color:#fff
```

### 📝 Descrição Detalhada do Fluxo

1. **Inicialização e Parâmetros**:
   - Carrega as configurações iniciais, incluindo credenciais, data de referência (`initDate`) e caminhos de rede.
   - Verifica o ambiente de execução (Produção vs. Teste).

2. **Controle de Estado (Checkpoints)**:
   - Verifica se a execução ocorre em um novo dia (`ifDayDiffers`). Se sim, limpa os registros de execução anterior no banco de dados para iniciar um novo ciclo de geração e envio.

3. **Geração do Arquivo (A450)**:
   - Verifica o status `cpGenerateA450`. Se pendente:
     - Realiza a conexão remota via GoGlobal/Citrix.
     - Autentica-se no **SGU** (Sistema de Gestão Unimed).
     - Navega até a rotina de exportação do PTU A450 e inicia a geração.
     - Atualiza o checkpoint no banco de dados para evitar reprocessamento.

4. **Obtenção do Arquivo (Download)**:
   - Verifica o status `cpGetA450`. Se pendente:
     - Acessa o módulo de relatórios/arquivos do sistema (SGU REP).
     - Identifica o arquivo gerado correspondente ao dia.
     - Realiza o download para o servidor local.
     - Compacta o arquivo (.zip) para otimizar o envio.
     - Encerra a sessão no GoGlobal.

5. **Envio e Finalização**:
   - Verifica o status `cpSendA450`. Se pendente:
     - Prepara e envia um e-mail contendo o arquivo ZIP do A450 para a lista de distribuição definida.
   - Executa a rotina de limpeza (`Transfer to backup`), movendo os arquivos gerados para a pasta histórica.
   - Dispara o e-mail final de notificação de sucesso.

### 📊 Entradas e Saídas
- **Entradas**:
  - Dados de conexão aos sistemas legados.
  - Parâmetros de data e identificação da Unimed.
- **Saídas**:
  - Arquivo PTU A450 (.zip) enviado por e-mail.
  - Registros de auditoria no banco de dados.
  - Estrutura de pastas organizada com backups diários.

### ⚠️ Regras de Negócio e Validações
- **Robustez**: O robô é desenhado para ser resiliente. Se falhar na etapa de geração, ao ser reiniciado, ele saberá que ainda precisa gerar. Se falhar no envio, ele pulará a geração (já feita) e tentará apenas enviar.
- **Segurança**: Credenciais são injetadas de forma segura e o acesso ao ambiente Citrix é fechado (`closeGoGlobal`) logo após o uso para liberar recursos.
- **Padronização**: Segue rigorosamente as especificações de nomenclatura e formato exigidos pelo padrão PTU da Unimed do Brasil.

### 🔗 Sistemas e Integrações
- **SGU**: Fonte dos dados dos serviços prestadores.
- **GoGlobal**: Camada de virtualização para acesso ao ERP.
- **Email Service**: Canal de entrega dos arquivos.

### 📈 Monitoramento
- Monitoramento via logs de execução no banco de dados.
- Alerta de erro via e-mail em caso de falha crítica (bloco `Try/Catch`).
- Confirmação visual via e-mail de "Sucesso" ao final do processo.
