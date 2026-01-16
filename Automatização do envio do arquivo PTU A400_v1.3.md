# 📋 Documentação de Processo: Automatização do envio do arquivo PTU A400

### 🎯 Visão Geral do Processo
Este robô automatiza o processo diário de geração, extração e envio do arquivo **PTU A400** (Movimentação Cadastral de Prestadores) para a Unimed. O arquivo A400 é um componente crítico do **Protocolo de Transação Unimed (PTU)**, regulado pela ANS, utilizado para manter atualizado o cadastro de prestadores de serviço (médicos, clínicas, hospitais) entre as unidades da Unimed.

O robô interage com múltiplos sistemas internos da Unimed (SGU, GIU, GoGlobal) para garantir que a movimentação cadastral seja processada e transmitida corretamente, mantendo a conformidade regulatória.

### 🎯 Objetivo e Escopo
- **O que automatiza**: 
  1. Geração do arquivo A400 no sistema SGU.
  2. Download e compactação do arquivo gerado.
  3. Envio do arquivo processado por e-mail para os responsáveis.
  4. Backup dos arquivos gerados.
- **Benefícios**: Elimina o trabalho manual repetitivo, reduz riscos de erro humano na geração de arquivos regulatórios e garante o cumprimento dos prazos diários de envio.
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
    
    subgraph "Processamento A400 (SGU/GIU)"
        TryStart --> GenCheck{Geração Concluída?}
        GenCheck -- Não --> LoginSGU[Login no SGU via GoGlobal]
        LoginSGU --> GenAction[Gerar Arquivo A400]
        GenAction --> SaveGen[Salvar Checkpoint: Gerado]
        SaveGen --> GetCheck
        GenCheck -- Sim --> GetCheck{Download Concluído?}
        
        GetCheck -- Não --> LoginREP[Acessar SGU REP]
        LoginREP --> Download[Baixar Arquivo A400]
        Download --> Zip[Compactar Arquivos (.zip)]
        Zip --> SaveGet[Salvar Checkpoint: Baixado]
        SaveGet --> SendCheck
        GetCheck -- Sim --> SendCheck{Envio Concluído?}
        
        SendCheck -- Não --> SendAction[Enviar Email com A400/A410]
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
   - O robô inicia carregando configurações como data de execução (`initDate`), número da Unimed (`unimedNumber`) e credenciais de acesso.
   - Verifica se está rodando em ambiente de Produção (`isProd`).

2. **Controle de Estado (Checkpoints)**:
   - Consulta o banco de dados para verificar o progresso do dia.
   - Se for um novo dia (`ifDayDiffers`), reseta os marcadores de progresso (Geração, Download e Envio) para garantir uma execução limpa.

3. **Geração do Arquivo (A400)**:
   - Verifica se o arquivo já foi gerado (`cpGenerateA400`). Se não:
     - Realiza login no ambiente Citrix/GoGlobal.
     - Acessa o sistema **SGU** (Sistema de Gestão Unimed).
     - Executa a rotina de geração do arquivo de movimentação cadastral.
     - Marca a etapa como concluída no banco de dados.

4. **Obtenção do Arquivo (Download)**:
   - Verifica se o arquivo já foi baixado (`cpGetA400`). Se não:
     - Acessa o módulo de Relatórios (SGU REP).
     - Localiza e baixa o arquivo gerado.
     - Compacta o arquivo em formato `.zip` para envio.
     - Fecha a conexão com o GoGlobal.

5. **Envio e Finalização**:
   - Verifica se o envio já foi realizado. Se não:
     - Envia um e-mail com os arquivos A400 (e A410, se aplicável) anexados para os destinatários configurados.
   - Move os arquivos processados para a pasta de backup (`Transfer to backup`), organizando o diretório de trabalho.
   - Envia um e-mail final de confirmação de sucesso (`finishEmail`).

### 📊 Entradas e Saídas
- **Entradas**:
  - Credenciais de acesso (SGU, GoGlobal, Email).
  - Configuração de diretórios para salvar arquivos temporários.
- **Saídas**:
  - Arquivo `.zip` contendo o PTU A400 enviado por e-mail.
  - Logs de execução e status atualizados no banco de dados do robô.
  - Arquivos armazenados na pasta de backup para histórico.

### ⚠️ Regras de Negócio e Validações
- **Controle de Duplicidade**: O uso de checkpoints (`cpGenerate`, `cpGet`, `cpSend`) impede que o robô repita etapas demoradas (como gerar o arquivo novamente) em caso de reinício ou falha parcial.
- **Tratamento de Erros**: Todo o fluxo principal é envolvido em um bloco de segurança (`Try/Catch`). Qualquer falha durante o acesso ao SGU ou manipulação de arquivos é capturada, registrada no banco de dados (`logError`) e encerra o robô de forma controlada.
- **Conformidade ANS**: O processo garante a geração diária do arquivo A400, essencial para a conformidade com as normas da ANS para troca de informações na saúde suplementar.

### 🔗 Sistemas e Integrações
- **SGU (Sistema de Gestão Unimed)**: Sistema core para geração dos arquivos.
- **GoGlobal/Citrix**: Plataforma de acesso remoto aos sistemas legados.
- **Outlook/SMTP**: Para envio dos arquivos processados.
- **Sistema de Arquivos**: Manipulação de pastas locais e rede para armazenamento e backup.

### 📈 Monitoramento
- O sucesso é confirmado pelo recebimento do e-mail final com o assunto indicando a conclusão do processamento do PTU A400.
- Em caso de falha, o banco de dados registra o erro específico, facilitando a atuação do suporte.
