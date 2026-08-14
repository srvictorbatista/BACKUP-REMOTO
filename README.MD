# CÓPIA DE BACKUP REMOTO PARA O HOST LOCAL

### COM AUDITORIA, ACOMPANHAMENTO E VERIFICAÇÃO DE INTEGRIDADE

<BR>

**AMBIENTE DE USO:** _SRV01 / SRV02 / SRV523 / SRV1809 / SRV3012 / SRV9266 / SRV-R-AFABB01 / SRV-BCO91 / MAINFRAME-PREVI01 / MAINFRAME-BB-CONVENIO02 / KAMAKISHIAXX / VENEROXX - bem como a infraestrutura de servidores isolados remotos e clusters sitiados em rede._

**AUTOR:** Sr. Victor Batista [GitHub](https://github.com/srvictorbatista) _- responsável técnico pela criação e manutenção deste documento e dos comandos nele descritos_

**DEMANDANTE:** Helo's World LTDA _[helosworld.com.br](https://helosworld.com.br/)_

**VERSÃO DESTE DOCUMENTO** _1.0.16 - corrige ciclo de vida do arquivo de status e uso do log, número de chamadas remotas e lista de dependências locais._

<BR><BR><BR>

### LIMITAÇÕES, POLÍTICAS E RESSALVAS

**A quem se destina a oferta pública deste documento?**
Ao time de R,D&S (Recrutamento, Desenvolvimento Humano e Seleção), a estudantes de gestão de infraestrutura de TI, DevOps e ao interesse público em geral. 

Diferente da publicação interna. Portanto, restrita por natureza. A oferta pública deste documento jamais e sob nenhuma hipótese deverá conter:

• Código interno de autenticação

• Número Interno Único (NIU/CDU) referente ao acervo da instituição demandante

• Credenciais como logins, senhas, endereços de rede ou servidores reais em uso


***_OBS: As orientações e práticas de que trata este documento seguem em progresso, até atingirem total conformidade com o o manual interno da demandante, com atualizações previstas em versões futuras. Havendo divergências entre esta documentação e o manual interno de boas práticas, o manual interno sempre prevalecerá._**
Este documento só poderá se tornar público, **somente após  6 meses (SEIS MESES)** à sua ultima atualização interna. Devendo haver uma clara distinção entre a versão em vigor internamente (de natureza, restrita) e a versão disponível publicamente, destinada a amplo estudo e treinamento. 
***Ao atingir total conformidade com o manual interno, este documento deverá ser considerado consolidado, não devendo ser disponibilizado publicamente. Em acordo com a política prevista em "segurança > política de ofuscamento" contida no manual citado. Não sendo de responsabilidade do autor, a publicação indevida, por ação de terceiros. Integrantes ou não da entidade demandante.**
<BR>

**RESSALVAS TÉCNICAS**
O nome do arquivo de backup remoto é assumido sem espaços em seu componente final, uma vez que `basename` opera sobre o resultado já tratado de `find`; caminhos remotos com espaços no diretório são suportados, pois `DIR_REMOTO` é tratado entre aspas na chamada ao `find`. A varredura remota considera apenas o primeiro nível do diretório de backups, por critério de projeto, e não percorre subdiretórios.

<BR><BR>

## OBSERVAÇÕES DE AUDITORIA

### SEGURANÇA E BOAS PRÁTICAS

A senha declarada em `SENHA` é entregue ao subprocesso por variável de ambiente, o que evita que apareça diretamente na linha de comando do processo `sshpass`; ainda assim, permanece em texto plano no histórico do shell durante o uso do comando de lançamento. Para uso recorrente ou em ambiente compartilhado, vale observar que a senha por si só é insuficiente sem as chaves SSH e os certificados de autorização de cada usuário/rede/tunnel para a efetiva autenticação por senha, mediantes os respectivos certificados de ambiente e chave SSH exigidos em cada conjunto tunnel/rede/servidor, isto não elimina a dependência de `sshpass` quanto qualquer segredo em texto plano na linha de comando. Com exceção da própria senha (sujeita a falhas de digitação por parte do usuário), a ausência de retorno em caso de erro de autenticação, ou verificação complementar sobre chaves e certificados no momento da execução ou aviso informativo. Não foi aplicada ao comando, a pedido deliberado da organização. Uma vez que os ambientes destinados (treinamento, homologação e produção). Quando em situações de falha, já são devidamente monitoradas e reportadas de forma ativa como ocorrências observacionais, independente de quaisquer comandos utilizados. 
Ainda assim, para limpeza eventual do histórico de comandos recomenda-se ao sair/desconectar, o uso de:

```bash
history -c && history -w && > ~/.bash_history && unset HISTFILE && exit   # LIMPAR HISTORICO DE COMANDOS E SAIR
```
_Em terminais BASH. O comando a cima apenas limpa o histórico do terminal de acesso (local ou remoto) encerrando a sessão. O que de forma alguma deve ser interpretado como isenção de monitoramento e registro das ações realizadas, já mencionados e inerentes à infraestrutura ao qual estes comandos se destinam._

Esta ação de limpeza é semelhante ao que o terminal proprietário da organização (Ministry-WS) já faz automaticamente. No entanto em terminais linux e unix padrão como TTY, SH e BASH. O histórico de comandos poderá eventualmente persistir. Por tanto, se você está fazendo uso destas orientações e comandos em um terminal diferente do terminal proprietário da demandante, recomenda-se realizar a limpeza (sanitização) ao deslogar. 

<BR>
<HR><BR><BR>

## INTRODUÇÃO

### OBJETIVO, CONTEXTO E FINALIDADE

Este documento descreve comandos de linha única, independentes de qualquer script instalado no sistema, responsáveis por copiar o backup mais recente de um serviço SCP isolado por um bastião SSH, hospedado em servidor remoto acessível via túnel privado, para o host local, com verificação de integridade por SHA-256 e progresso de transferência em tempo real. O primeiro é o comando condençado contendo em um único envio, o lançamento e acompanhamento instantâneo. O segundo comando lança a cópia em segundo plano, sobrevivendo ao encerramento da sessão SSH que o originou; O terceiro localiza e acompanha, em tempo real, o log da execução correspondente. Ambos operam sobre a mesma convenção de nomenclatura, o que permite reutilizar exatamente os mesmos comandos em quaisquer servidor, inclusive com múltiplas cópias simultâneas, alterando apenas as variáveis declaradas no início de cada linha.

<BR>

## JUSTIFICATIVA

### MOTIVAÇÃO TÉCNICA

A cópia manual de backups remotos, quando feita por comandos avulsos digitados em sessão interativa, está sujeita a interrupção por queda de conexão, ausência de registro do que foi feito, e nenhuma garantia de que o arquivo chegou íntegro ao destino. A consolidação de todo o processo em um único comando autocontido resolve essas três fragilidades ao mesmo tempo: torna a execução resiliente à sessão que a originou, registra cada etapa em log próprio e acompanhável, e condiciona o sucesso da operação à comparação criptográfica entre origem e destino, e não apenas à ausência de erro de rede.

<BR><BR>

## O QUE OS COMANDOS ASSEGURAM

### GARANTIAS OPERACIONAIS

A execução do backup não depende do terminal que a iniciou permanecer aberto, pois o processo é desvinculado da sessão por `nohup` e `disown`. A integridade do arquivo copiado é assegurada pela comparação do SHA-256 calculado independentemente no host remoto e no host local, e não pela simples ausência de erro na transferência. A observabilidade do processo é assegurada por um log único por execução, acompanhável em tempo real e com progresso percentual real da transferência. A limpeza do ambiente é assegurada pela remoção automática do arquivo de status ao término da execução, com ou sem sucesso, evitando acúmulo de semáforos órfãos; o log em si permanece no destino após a execução, disponível para consulta posterior. A reutilização para outros servidores é assegurada pela total independência do comando em relação a qualquer valor fixo: nome do servidor, credenciais e diretórios são declarados como variáveis no início da própria linha de comando.

<BR>

## ALGORITMO DE CÓPIA REMOTA

### CRITÉRIOS E SEQUÊNCIA DE EXECUÇÃO

O comando de lançamento inicia registrando, via `trap`, a remoção do arquivo de status (semáforo `.running`) como última ação do processo, e ativando `pipefail`, condição necessária para que uma falha de conexão durante a transferência via pipe seja corretamente detectada, já que sem essa opção o status de saída do pipeline refletiria apenas o do último comando da cadeia. Em seguida, toda a saída do processo passa a ser redirecionada simultaneamente para a tela e para o arquivo de log. A primeira ação efetiva é a localização, no diretório remoto configurado, do arquivo com extensão `.tar.xz` de modificação mais recente, critério que assume que o backup mais novo é sempre o desejado. Confirmada a existência desse arquivo, seu tamanho exato é obtido no host remoto, valor necessário para que a etapa seguinte calcule percentual de progresso real, e não uma estimativa. A transferência ocorre então por leitura remota do arquivo, encadeada diretamente à gravação local, com a barra de progresso exibindo percentual, volume transferido, taxa média e tempo decorrido durante todo o percurso. Concluída a cópia, o critério de aceite final é a igualdade entre o hash SHA-256 calculado no arquivo de origem, no host remoto, e o hash calculado no arquivo recém-copiado, no host local; qualquer divergência, assim como qualquer falha na etapa de transferência, resulta na remoção imediata do arquivo local e no encerramento do processo com erro, nunca deixando um backup incompleto ou não verificado no destino.

<BR>

## ALGORITMO DE ACOMPANHAMENTO

### CRITÉRIOS E SEQUÊNCIA DE EXECUÇÃO

O critério utilizado é exclusivamente a existência de um arquivo de log correspondente ao identificador do servidor informado. A busca ordena, por data de modificação, todos os arquivos que correspondem ao padrão de nomenclatura combinando o diretório configurado com o identificador do servidor, e seleciona o mais recente entre eles, critério que privilegia a execução em andamento ou a mais próxima do momento atual sobre qualquer resquício de execução anterior. Encontrado o log, o acompanhamento passa a exibir seu conteúdo em tempo real, incluindo as mensagens de status e a barra de progresso gerada pela etapa de transferência do comando de lançamento. Na ausência de qualquer arquivo correspondente, o comando informa objetivamente que não há backup em andamento para aquele identificador, sem ambiguidade quanto a erro de execução ou simples inexistência de processo ativo.

<BR><BR>

## PRÉ-REQUISITOS

### DEPENDÊNCIAS LOCAIS E REMOTAS

No host remoto (onde o backup é gerado), são necessários os utilitários `find`, `stat`, `sha256sum` e `cat`, além de permissão de `sudo` sem interação para o usuário utilizado na conexão. No host local (onde os comandos são executados), são necessários `ssh`, `sshpass`, `sha256sum`, `numfmt`, `tee`, `printf` e o conjunto padrão de utilitários coreutils, além de `PV`, responsável por exibir o progresso real da transferência. Caso o utilitário `PV` não esteja instalado, a instalação é feita com o comando abaixo. 

```bash
apt-get update && apt-get install -y pv
```

<BR>

**VERIFICAÇÃO RÁPIDA DE TODAS AS DEPENDÊNCIAS**

O mesmo comando serve para verificação em ambos os hosts (local e remoto):

```bash
D_LOCAL="ssh sshpass sha256sum numfmt tee printf pv find stat cat sudo"; FALHA=0; printf '\n%-12s %-10s %-45s %s\n' "AMBIENTE" "STATUS" "DEPENDENCIA" "ACAO NECESSARIA"; printf '%-12s %-10s %-45s %s\n' "------------" "----------" "---------------------------------------------" "---------------------------------------------"; for C in $D_LOCAL; do if command -v "$C" >/dev/null 2>&1; then printf '%-12s %-10s %-45s %s\n' "HOST ATUAL" "OK" "$C ($(command -v "$C"))" "-"; else case "$C" in ssh) ACAO="apt-get update && apt-get install -y openssh-client";; sshpass) ACAO="(opcional no host remoto): apt-get update && apt-get install -y sshpass";; pv) ACAO="apt-get update && apt-get install -y pv";; sudo) ACAO="apt-get update && apt-get install -y sudo";; *) ACAO="apt-get update && apt-get install -y coreutils findutils";; esac; printf '\033[38;5;208m%-12s %-10s %-45s %s\033[0m\n' "HOST ATUAL" "AUSENTE" "$C" "$ACAO"; FALHA=1; fi; done; if command -v sudo >/dev/null 2>&1; then sudo -n true >/dev/null 2>&1 && printf '%-12s %-10s %-45s %s\n' "HOST ATUAL" "OK" "sudo -n (SEM INTERACAO)" "-" || { printf '\033[38;5;208m%-12s %-10s %-45s %s\033[0m\n' "HOST ATUAL" "PENDENTE" "sudo -n (REQUER CONFIGURACAO)" "(opcional no host local): sudo visudo # CONFIGURAR PERMISSAO NOPASSWD"; FALHA=1; }; fi; printf '\nRESULTADO: '; [ "$FALHA" -eq 0 ] && printf '\033[37;42m TODAS AS DEPENDENCIAS ESTAO DISPONIVEIS. \033[0m\n\n\n' || printf '\033[37;41m EXISTEM DEPENDENCIAS AUSENTES OU PENDENTES. \033[0m\n\n\n'; # VERIFICA AS DEPENDENCIAS DO HOST LOCAL OU REMOTO
```

OBS: Caso identifique alguma configuração divergente ou dependência ausente no servidor remoto. Pertencente a uma das infraestruturas informadas, antes da execução dos comandos a seguir. Contate o administrador da infraestrutura remota ou parceiro responsável. Pois cada ambiente/servidor responde por uma prática e política de implementação própria, contida no manual técnico interno. E foge do foco desta documentação. 

Se você também é o administrador da infraestrutura remota e está de posse do seu manual interno. Havendo divergências (técnicas ou de governança) siga conforme já descrito na sessão **LIMITAÇÕES, POLÍTICAS E RESSALVAS** devendo sempre prevalecer os procedimentos descritos em seu manual. 

<BR>

## CONFIGURAÇÃO

### VARIÁVEIS AJUSTÁVEIS

Todos os comandos declaram suas variáveis no início da própria linha, e nenhum outro trecho precisa ser alterado para reutilizá-los em outro servidor. `NOME` identifica logicamente o servidor, define o nome do arquivo de log e é o elo que associa o comando de lançamento ao comando de acompanhamento correspondente; o mesmo valor deve ser usado em ambos. `SSH_ALVO`, presente apenas no lançamento, define o destino SSH completo, no formato usuário-dispositivo-relay próprio do túnel SSH/SCP. `SENHA`, também exclusiva do lançamento, é a credencial de autenticação, entregue ao `ssh` através da variável de ambiente `SSHPASS`, consumida pelo `sshpass -e`. `DIR_REMOTO` e `DIR_LOCAL` definem, respectivamente, o diretório onde os arquivos de backup são gravados no host remoto e o diretório de destino da cópia no host local. `DIR_LOGS`, exclusiva do acompanhamento, define o diretório onde os logs são procurados, com `/tmp` como padrão.

Havendo necessidade. As variáveis poderão ser informadas a cada execução no inicio do comando ou temporariamente redefinidas no ambiente (sessão do terminal):  
```bash
NOME="NOME_SERVIDOR"; SENHA="SUA_SENHA_AQUI"; SSH_ALVO="SEU_USER@HOSTDESTINO"; # REDEFINE VARIAVEIS DE AMBIENTE TEMPORARIAMENTE
```
INFO: Ao deslogar, estas definições retornam ao padrão do ambiente. E quando implementadas, servem apenas ao terminal em uso.



<BR>



## COMANDOS CONDENSADOS (LANÇAMENTO E ACOMPANHAMENTO)

### REALIZA A CÓPIA JÁ COM ACOMPANHAMENTO E VALIDAÇÃO EM TEMPO REAL

```bash
NOME="${NOME:-NOME_SERVIDOR}"; SENHA="${SENHA:-SUA_SENHA_AQUI}"; SSH_ALVO="${SSH_ALVO:-SEU_USER@HOSTDESTINO}"; DIR_REMOTO="${DIR_REMOTO:-/lxc/backup}"; DIR_LOCAL="${DIR_LOCAL:-/lxc/backup}"; LOG="/tmp/backup-remoto-${NOME}-$(date +%Y%m%d-%H%M%S).log"; STATUS="/tmp/backup-remoto-${NOME}.running"; if [ -e "$STATUS" ] && kill -0 "$(cat "$STATUS" 2>/dev/null)" 2>/dev/null; then printf '\n \033[97;43m Já existe um backup em andamento para %s (PID %s). Aguarde a conclusão. \033[0m\n Ou acompanhe o log em\n "tail -f %s"\n\n' "$NOME" "$(cat "$STATUS")" "$LOG"; else rm -f "$STATUS"; if [ "$NOME" = "NOME_SERVIDOR" ]; then printf '\n \033[97;41m ERRO: defina a variável NOME antes de executar novamente. \033[0m\n\n'; elif [ "$SENHA" = "SUA_SENHA_AQUI" ]; then printf '\n \033[97;41m ERRO: defina a variável SENHA antes de executar novamente. \033[0m\n\n'; elif [ "$SSH_ALVO" = "SEU_USER@HOSTDESTINO" ]; then printf '\n \033[97;41m ERRO: defina a variável SSH_ALVO antes de executar novamente. \033[0m\n\n'; elif [ ! -d "$DIR_LOCAL" ] || [ ! -w "$DIR_LOCAL" ]; then printf '\n \033[97;41m ERRO: %s não existe ou não é gravável. \033[0m\n\n' "$DIR_LOCAL"; elif ! command -v pv >/dev/null 2>&1; then printf '\n \033[97;41m ERRO: PV é necessário, antes de repetir este comando \033[0m \n execute "apt-get update && apt-get install -y pv" \n\n'; else nohup env SSHPASS="$SENHA" ALVO="$SSH_ALVO" DIR_REMOTO="$DIR_REMOTO" DIR_LOCAL="$DIR_LOCAL" LOG="$LOG" STATUS="$STATUS" bash -c 'set -o pipefail; exec > >(tee -a "$LOG") 2>&1; ERRO="\033[97;41m"; VERDE="\033[97;42m"; RESET="\033[0m"; trap "rm -f \"$STATUS\"" EXIT; echo " [$(date "+%F %T")] Iniciando processo de copia do backup remoto"; echo " [$(date "+%F %T")] Verificando credenciais..."; sshpass -e ssh -o ConnectTimeout=15 -o StrictHostKeyChecking=accept-new "$ALVO" "true" || { printf " %b[$(date "+%F %T")] ERRO: falha de conexão SSH com %s %b \n\n\n" "$ERRO" "$ALVO" "$RESET"; exit 1; }; echo " [$(date "+%F %T")] Conexão SSH OK"; LATEST=$(sshpass -e ssh -o ConnectTimeout=15 -o StrictHostKeyChecking=accept-new "$ALVO" "sudo -n find \"$DIR_REMOTO\" -maxdepth 1 -type f -name \"*.tar.xz\" -printf \"%T@ %p\n\" | sort -nr | head -1 | cut -d\" \" -f2-") || { printf " %b[$(date "+%F %T")] ERRO: falha ao localizar backup remoto, verifique sudo sem senha no dispositivo remoto %b \n\n\n" "$ERRO" "$RESET"; exit 1; }; [ -n "$LATEST" ] || { printf " %b[$(date "+%F %T")] ERRO: backup não encontrado %b \n\n\n" "$ERRO" "$RESET"; exit 1; }; SIZE=$(sshpass -e ssh -o ConnectTimeout=15 -o StrictHostKeyChecking=accept-new "$ALVO" "sudo -n stat -c %s \"$LATEST\"") || { printf " %b[$(date "+%F %T")] ERRO: não foi possível obter o tamanho do backup %b \n\n\n" "$ERRO" "$RESET"; exit 1; }; AVAIL=$(df --output=avail -B1 "$DIR_LOCAL" | tail -1); [ "$AVAIL" -ge "$SIZE" ] || { printf " %b[$(date "+%F %T")] ERRO: espaço insuficiente em %s, livre %s, necessário %s %b \n\n\n" "$ERRO" "$DIR_LOCAL" "$(numfmt --to=iec "$AVAIL")" "$(numfmt --to=iec "$SIZE")" "$RESET"; exit 1; }; DEST="$DIR_LOCAL/$(basename "$LATEST")"; echo " [$(date "+%F %T")] Copiando: $LATEST"; echo -e " [$(date "+%F %T")] Tamanho: $(numfmt --to=iec "$SIZE") \n"; sshpass -e ssh -o ConnectTimeout=15 -o StrictHostKeyChecking=accept-new "$ALVO" "sudo -n cat \"$LATEST\"" | pv -fpterab -s "$SIZE" > "$DEST" || { printf " %b[$(date "+%F %T")] ERRO: falha na transferência %b \n\n\n" "$ERRO" "$RESET"; rm -f "$DEST"; exit 1; }; echo " [$(date "+%F %T")] Verificando integridade..."; REMOTE_SHA=$(sshpass -e ssh -o ConnectTimeout=15 -o StrictHostKeyChecking=accept-new "$ALVO" "sudo -n sha256sum \"$LATEST\" | cut -d\" \" -f1") || { printf " %b[$(date "+%F %T")] ERRO: não foi possível calcular SHA256 remoto %b \n\n\n" "$ERRO" "$RESET"; rm -f "$DEST"; exit 1; }; LOCAL_SHA=$(sha256sum "$DEST" | cut -d" " -f1) || { printf " %b[$(date "+%F %T")] ERRO: não foi possível calcular SHA256 local %b \n\n\n" "$ERRO" "$RESET"; rm -f "$DEST"; exit 1; }; echo " [$(date "+%F %T")] SHA256 remoto: $REMOTE_SHA"; echo " [$(date "+%F %T")] SHA256 local:  $LOCAL_SHA"; [ "$REMOTE_SHA" = "$LOCAL_SHA" ] || { printf " %b[$(date "+%F %T")] ERRO: integridade inválida %b \n\n\n" "$ERRO" "$RESET"; rm -f "$DEST"; exit 1; }; echo " [$(date "+%F %T")] Integridade OK: SHA256=$LOCAL_SHA"; printf " %b CONCLUIDO COM SUCESSO! %b \n\n\n" "$VERDE" "$RESET"; echo " [$(date "+%F %T")] Backup concluído: $DEST"; ls -lh "$DEST"' _ </dev/null >/dev/null 2>&1 & PID=$!; echo "$PID" > "$STATUS"; tail -n +1 -F "$LOG" 2>/dev/null & TAIL_PID=$!; while [ -e "$STATUS" ] && kill -0 "$PID" 2>/dev/null; do sleep 1; done; sleep 1; kill "$TAIL_PID" 2>/dev/null; wait "$TAIL_PID" 2>/dev/null; wait "$PID" 2>/dev/null; printf '\n\033[38;5;81m [$(date "+%F %T")] Acompanhamento finalizado: %s \033[0m\n\n\n' "$LOG"; fi; fi # COPIA BACKUP REMOTO PARA O HOST ATUAL COM ACOMPANHAMENTO E VALIDAÇÃO
```
Busca se as variáveis necessárias já estão presentes no ambiente. Não encontrando, define nome, senha, servidor remoto e pastas, cria um arquivo de registro e verifica se já existe outra cópia em andamento. já havendo um pedido para o mesmo servidor em andamento aborda a execução, evitando concorrência e/ou corrupção dos dados em progresso. Após esta primeira validação, confirma se a pasta local existe e permite gravação, e se o programa `pv`, responsável por mostrar o progresso, está instalado e disponível.

Em seguida, valida as credenciais informadas e a conexão ao servidor remoto. Sendo bem sucedido nesta etapa, realiza a conexão e procura o arquivo `.tar.xz` mais recentemente por data de modificação na pasta remota informada, levanta seu tamanho e verifica se há espaço suficiente no host local para a recepção do arquivo encontrado. Só após todas as verificações anteriores resultarem de forma positiva, inicia a transferência mostrando continuamente o percentual, velocidade, quantidade transferida, enquanto grava tudo em um arquivo de registro.

Ao terminar a transferência, faz uma auditoria de segurança com SHA-256 com base na hash obtida a partir do arquivo original e a confronta com a versão obtida da transferência. Se forem iguais, confirma a integridade e informa a conclusão. Havendo divergência, considera a cópia inválida, apaga o arquivo recebido e registra o erro.

Durante toda a operação, o comando mantém um sinalizador de estado como aberto. Ao finalizar a operação (com sucesso ou falha), remove esse identificador, encerra o acompanhamento e informa onde está o registro final. Exibindo algo semelhante a:

```
 [2025-11-02 18:15:16] Verificando credenciais...
 [2025-11-02 18:15:22] Copiando: /lxc/backup/SERVIDOR-backup-2025-11-02.tar.xz
 [2025-11-02 18:15:22] Tamanho: 2480G

 2480.1GiB 1:06:36 [983.20MiB/s] (1070.20MiB/s) [====================>] 100%

Acompanhamento finalizado.
```


<BR>



## LANÇAMENTO (COMANDO PRINCIPAL)

### CÓPIA EM SEGUNDO PLANO COM VALIDAÇÃO
Se preferir apenas lançar todo o processo em segundo plano, para posterior acompanhamento. Basta realizar somente a etapa de lançamento com o seguinte comando:

```bash
NOME="${NOME:-NOME_SERVIDOR}"; SENHA="${SENHA:-SUA_SENHA_AQUI}"; SSH_ALVO="${SSH_ALVO:-SEU_USER@HOSTDESTINO}"; DIR_REMOTO="${DIR_REMOTO:-/lxc/backup}"; DIR_LOCAL="${DIR_LOCAL:-/lxc/backup}"; LOG="/tmp/backup-remoto-${NOME}-$(date +%Y%m%d-%H%M%S).log"; STATUS="/tmp/backup-remoto-${NOME}.running"; if [ -e "$STATUS" ] && kill -0 "$(cat "$STATUS" 2>/dev/null)" 2>/dev/null; then printf '\n \033[97;43m Já existe um backup em andamento para %s (PID %s). Aguarde a conclusão. \033[0m\n\n' "$NOME" "$(cat "$STATUS")"; else rm -f "$STATUS"; if [ "$SENHA" = "SUA_SENHA_AQUI" ]; then printf '\n \033[97;41m ERRO: defina a variável SENHA antes de executar. \033[0m\n\n'; elif [ ! -d "$DIR_LOCAL" ] || [ ! -w "$DIR_LOCAL" ]; then printf '\n \033[97;41m ERRO: %s não existe ou não é gravável. \033[0m\n\n' "$DIR_LOCAL"; elif ! command -v pv >/dev/null 2>&1; then printf '\n \033[97;41m ERRO: PV é necessário, antes de repetir este comando \033[0m \n execute "apt-get update && apt-get install -y pv" \n\n'; else nohup env SSHPASS="$SENHA" ALVO="$SSH_ALVO" DIR_REMOTO="$DIR_REMOTO" DIR_LOCAL="$DIR_LOCAL" LOG="$LOG" STATUS="$STATUS" bash -c 'set -o pipefail; exec > >(tee -a "$LOG") 2>&1; ERRO="\033[97;41m"; VERDE="\033[97;42m"; RESET="\033[0m"; trap "rm -f \"$STATUS\"" EXIT; echo " [$(date "+%F %T")] Iniciando processo de copia do backup remoto"; echo " [$(date "+%F %T")] Verificando credenciais..."; sshpass -e ssh -o ConnectTimeout=15 -o StrictHostKeyChecking=accept-new "$ALVO" "true" || { printf " %b[$(date "+%F %T")] ERRO: falha de conexão SSH com %s %b \n\n\n" "$ERRO" "$ALVO" "$RESET"; exit 1; }; echo " [$(date "+%F %T")] Conexão SSH OK"; LATEST=$(sshpass -e ssh -o ConnectTimeout=15 -o StrictHostKeyChecking=accept-new "$ALVO" "sudo -n find \"$DIR_REMOTO\" -maxdepth 1 -type f -name \"*.tar.xz\" -printf \"%T@ %p\n\" | sort -nr | head -1 | cut -d\" \" -f2-") || { printf " %b[$(date "+%F %T")] ERRO: falha ao localizar backup remoto, verifique sudo sem senha no dispositivo remoto %b \n\n\n" "$ERRO" "$RESET"; exit 1; }; [ -n "$LATEST" ] || { printf " %b[$(date "+%F %T")] ERRO: backup não encontrado %b \n\n\n" "$ERRO" "$RESET"; exit 1; }; SIZE=$(sshpass -e ssh -o ConnectTimeout=15 -o StrictHostKeyChecking=accept-new "$ALVO" "sudo -n stat -c %s \"$LATEST\"") || { printf " %b[$(date "+%F %T")] ERRO: não foi possível obter o tamanho do backup %b \n\n\n" "$ERRO" "$RESET"; exit 1; }; AVAIL=$(df --output=avail -B1 "$DIR_LOCAL" | tail -1); [ "$AVAIL" -ge "$SIZE" ] || { printf " %b[$(date "+%F %T")] ERRO: espaço insuficiente em %s, livre %s, necessário %s %b \n\n\n" "$ERRO" "$DIR_LOCAL" "$(numfmt --to=iec "$AVAIL")" "$(numfmt --to=iec "$SIZE")" "$RESET"; exit 1; }; DEST="$DIR_LOCAL/$(basename "$LATEST")"; echo " [$(date "+%F %T")] Copiando: $LATEST"; echo -e " [$(date "+%F %T")] Tamanho: $(numfmt --to=iec "$SIZE") \n"; sshpass -e ssh -o ConnectTimeout=15 -o StrictHostKeyChecking=accept-new "$ALVO" "sudo -n cat \"$LATEST\"" | pv -fpterab -s "$SIZE" > "$DEST" || { printf " %b[$(date "+%F %T")] ERRO: falha na transferência %b \n\n\n" "$ERRO" "$RESET"; rm -f "$DEST"; exit 1; }; echo " [$(date "+%F %T")] Verificando integridade..."; REMOTE_SHA=$(sshpass -e ssh -o ConnectTimeout=15 -o StrictHostKeyChecking=accept-new "$ALVO" "sudo -n sha256sum \"$LATEST\" | cut -d\" \" -f1") || { printf " %b[$(date "+%F %T")] ERRO: não foi possível calcular SHA256 remoto %b \n\n\n" "$ERRO" "$RESET"; rm -f "$DEST"; exit 1; }; LOCAL_SHA=$(sha256sum "$DEST" | cut -d" " -f1) || { printf " %b[$(date "+%F %T")] ERRO: não foi possível calcular SHA256 local %b \n\n\n" "$ERRO" "$RESET"; rm -f "$DEST"; exit 1; }; echo " [$(date "+%F %T")] SHA256 remoto: $REMOTE_SHA"; echo " [$(date "+%F %T")] SHA256 local:  $LOCAL_SHA"; [ "$REMOTE_SHA" = "$LOCAL_SHA" ] || { printf " %b[$(date "+%F %T")] ERRO: integridade inválida %b \n\n\n" "$ERRO" "$RESET"; rm -f "$DEST"; exit 1; }; echo " [$(date "+%F %T")] Integridade OK: SHA256=$LOCAL_SHA"; printf " %b CONCLUIDO COM SUCESSO! %b \n\n\n" "$VERDE" "$RESET"; echo " [$(date "+%F %T")] Backup concluído: $DEST"; ls -lh "$DEST"' _ </dev/null >/dev/null 2>&1 & echo $! > "$STATUS"; disown; fi; fi # COPIA BACKUP REMOTO PARA O HOST ATUAL COM VERIFICAÇÃO
```

O caractere isolado `_`, logo após o fechamento das aspas simples do script, não é um erro de digitação e corresponde ao `$0` posicional (argumento de auto referência) exigido pela sintaxe `bash -c 'script' $0`;  Não devendo ser removido. Pois serve como identificador do processo em listagens como `ps`.

_INFO: Há diversas formas de se executar um script ou comando oculto da listagem de PID do sistema. Além desta não ser a nossa intenção, omitir este identificador de argumento, apenas nos impossibilitaria de ter um número para acompanhamento do processo (previsto no comando secundário, seguinte). Que por sua vez, continuaria sendo executado em segundo plano, e observado pelo sistema de monitoramento e observabilidade até sua conclusão. Detalhes sobre os recursos de observabilidade presentes em cada infraestrutura, ultrapassam o nível de informação pretendida neste documento. Especialmente em sua versão destinada ao amplo público, mas abordados (podendo ser questionados e verificados de forma livre) durante os treinamentos internos da demandante._

<BR>

## ACOMPANHAMENTO (COMANDO SECUNDÁRIO)

### LOCALIZAÇÃO E LEITURA DO LOG EM TEMPO REAL
Além de um acompanhamento em momento posterior ao lançamento. Este comando é especialmente útil quando se tem apenas as credenciais locais. Ou não é desejável dispor de suas credenciais remotas. Além de possibilitar o acompanhamento do mesmo processo em diversos terminais ligados ao mesmo host:

```bash
NOME="${NOME:-NOME_SERVIDOR}"; DIR_LOGS="/tmp"; STATUS="/tmp/backup-remoto-${NOME}.running"; LOG_ATUAL=$(ls -t "${DIR_LOGS}/backup-remoto-${NOME}-"*.log 2>/dev/null | head -1); if [ -e "$STATUS" ] && kill -0 "$(cat "$STATUS" 2>/dev/null)" 2>/dev/null; then printf '\n\033[38;5;81mAcompanhando: %s \033[0m\n\n' "$LOG_ATUAL"; DETACHED=0; trap 'DETACHED=1' INT; if [ -n "$LOG_ATUAL" ]; then tail -n +1 -F "$LOG_ATUAL" & TAIL_PID=$!; while [ "$DETACHED" -eq 0 ] && [ -e "$STATUS" ] && kill -0 "$(cat "$STATUS" 2>/dev/null)" 2>/dev/null; do sleep 1; done; kill "$TAIL_PID" 2>/dev/null; wait "$TAIL_PID" 2>/dev/null; else while [ "$DETACHED" -eq 0 ] && [ -e "$STATUS" ] && kill -0 "$(cat "$STATUS" 2>/dev/null)" 2>/dev/null; do LOG_ATUAL=$(ls -t "${DIR_LOGS}/backup-remoto-${NOME}-"*.log 2>/dev/null | head -1); [ -n "$LOG_ATUAL" ] && break; sleep 1; done; if [ -n "$LOG_ATUAL" ] && [ "$DETACHED" -eq 0 ]; then tail -n +1 -F "$LOG_ATUAL" & TAIL_PID=$!; while [ "$DETACHED" -eq 0 ] && [ -e "$STATUS" ] && kill -0 "$(cat "$STATUS" 2>/dev/null)" 2>/dev/null; do sleep 1; done; kill "$TAIL_PID" 2>/dev/null; wait "$TAIL_PID" 2>/dev/null; fi; fi; trap - INT; if [ "$DETACHED" -eq 1 ]; then printf '\n \033[38;5;81m Exibição encerrada. O acompanhamento foi interrompido, mas o backup remoto continua em segundo plano. \033[0m\n \033[38;5;81m Execute novamente este comando para retomar a exibição. \033[0m\n\n\n'; else printf '\n\033[38;5;81m [$(date "+%F %T")] Acompanhamento finalizado.\033[0m\n\n\n'; fi; else rm -f "$STATUS"; printf '\n  \033[97;43m Nenhum backup remoto em andamento. \033[0m\n'; [ -n "$LOG_ATUAL" ] && printf '  \033[38;5;130mÚltimo log disponível: %s\033[0m\n\n\n' "$LOG_ATUAL"; fi # ACOMPANHAR A TRANSFERENCIA E PROGRESSO DO BACKUP REMOTO EM TEMPO REAL, COM SAÍDA VIA CTRL+C SEM INTERROMPER O BACKUP
```

<BR> 

**COMO CONFIRMAR IMEDIATAMENTE**

Em terminal de acompanhamento, após as validações iniciais normalmente aparece a linha de progresso:
```
2480.0GiB 0:12:34 [986.5MiB/s] [==============================>] 100% ETA 0:00:00
```
Esta linha de progresso é fornecida pelo utilitário PV. Sua exibição ou monitoramento não são obrigatórios. Podendo a sessão do terminal ser fechada sem prejuízos, enquanto a transferência continuará normalmente em segundo plano. 

<BR>

### CONVENÇÃO DE NOMENCLATURA E CICLO DE VIDA DE LOGs

**PADRÃO DE IDENTIFICAÇÃO**

```
/tmp/backup-remoto-SERVIDOR01-20251102-221530.log
/tmp/backup-remoto-SERVIDOR02-20251102-221602.log
/tmp/backup-remoto-SERVIDOR03-20251102-221711.log

```

O nome do servidor compõe a identidade lógica da execução, e o timestamp diferencia execuções consecutivas do mesmo servidor. O arquivo de log é criado antes da primeira escrita, recebe toda a saída do processo e permanece no destino após o término da execução, com sucesso ou falha, disponível para auditoria posterior. É o arquivo de status (`.running`) associado ao mesmo `NOME`, e não o log, que é removido pelo `trap` ao término, liberando o identificador para uma nova execução e permitindo múltiplas execuções simultâneas para servidores distintos, cada uma com seu próprio comando de acompanhamento, sem qualquer interferência entre os diversos fluxos simultâneos possíveis.

<BR><BR><BR>

## TRATAMENTO DE ERROS

### CONDIÇÕES DE FALHA E AÇÕES CORRETIVAS

O script encerra imediatamente, sem prosseguir para as etapas seguintes, diante de falha de conexão em qualquer uma das cinco verificações remotas (verificação de credenciais, localização do arquivo, obtenção do tamanho, leitura para transferência e cálculo do SHA-256), ausência de arquivo `.tar.xz` no diretório remoto, espaço insuficiente no destino para receber o arquivo, falha na transferência detectada graças a `pipefail`, ou divergência entre os hashes SHA-256 remoto e local. Nas duas últimas condições, o arquivo local parcialmente copiado é removido antes do encerramento, de modo que um backup incompleto ou corrompido nunca permaneça no destino. Evitando desperdício de armazenamento de forma automática.

<BR>

## CONSIDERAÇÕES FINAIS

### PARALELISMO POR SERVIDOR

Como o identificador `NOME` determina tanto o arquivo de log quanto, indiretamente, o arquivo de destino, múltiplos backups para servidores diferentes podem ser lançados em paralelo sem colisão, desde que cada execução concorrente utilize um `NOME` exclusivo. O acompanhamento de cada execução é feito repetindo o comando de acompanhamento, em terminais separados, com o `NOME` correspondente.

<BR><BR><BR>

### COMANDOS ÚTEIS E ADICIONAIS


<BR>

Verificação rápida de todas as dependências (locais e remotas):

```bash
D_LOCAL="ssh sshpass sha256sum numfmt tee printf pv find stat cat sudo"; FALHA=0; printf '\n%-12s %-10s %-45s %s\n' "AMBIENTE" "STATUS" "DEPENDENCIA" "ACAO NECESSARIA"; printf '%-12s %-10s %-45s %s\n' "------------" "----------" "---------------------------------------------" "---------------------------------------------"; for C in $D_LOCAL; do if command -v "$C" >/dev/null 2>&1; then printf '%-12s %-10s %-45s %s\n' "HOST ATUAL" "OK" "$C ($(command -v "$C"))" "-"; else case "$C" in ssh) ACAO="apt-get update && apt-get install -y openssh-client";; sshpass) ACAO="(opcional no host remoto): apt-get update && apt-get install -y sshpass";; pv) ACAO="apt-get update && apt-get install -y pv";; sudo) ACAO="apt-get update && apt-get install -y sudo";; *) ACAO="apt-get update && apt-get install -y coreutils findutils";; esac; printf '\033[38;5;208m%-12s %-10s %-45s %s\033[0m\n' "HOST ATUAL" "AUSENTE" "$C" "$ACAO"; FALHA=1; fi; done; if command -v sudo >/dev/null 2>&1; then sudo -n true >/dev/null 2>&1 && printf '%-12s %-10s %-45s %s\n' "HOST ATUAL" "OK" "sudo -n (SEM INTERACAO)" "-" || { printf '\033[38;5;208m%-12s %-10s %-45s %s\033[0m\n' "HOST ATUAL" "PENDENTE" "sudo -n (REQUER CONFIGURACAO)" "(opcional no host local): sudo visudo # CONFIGURAR PERMISSAO NOPASSWD"; FALHA=1; }; fi; printf '\nRESULTADO: '; [ "$FALHA" -eq 0 ] && printf '\033[37;42m TODAS AS DEPENDENCIAS ESTAO DISPONIVEIS. \033[0m\n\n\n' || printf '\033[37;41m EXISTEM DEPENDENCIAS AUSENTES OU PENDENTES. \033[0m\n\n\n'; # VERIFICA AS DEPENDENCIAS DO HOST LOCAL OU REMOTO
```

<BR>

As variáveis poderão ser informadas a cada execução, no inicio do comando ou temporariamente redefinidas na sessão do terminal:  
```bash
NOME="NOME_SERVIDOR"; SENHA="SUA_SENHA_AQUI"; SSH_ALVO="SEU_USER@HOSTDESTINO"; # REDEFINE VARIAVEIS DE AMBIENTE TEMPORARIAMENTE
```
Ao deslogar, estas definições temporárias serão perdidas...

<BR>

Encerrar um processo em execução de forma imediata e com verificação rápida em uma linha:
```bash
kill <PID_NUM> && kill -0 <PID_NUM> 2>/dev/null && echo "PROCESSO AINDA ATIVO" || echo -e "PROCESSO ENCERRADO! \n" # ENCERRA PID COM VERIFICAÇÃO
```
<BR>

Verificar arquivos na pasta de destino (opcional):
```bash
ls  -lh /lxc/backup # VERIFICAR PASTA DE DESTINO
```

<BR>
<BR>

Acompanhar progresso da transferência via watch (opcional):
```bash
watch -n  2  'ls -lh /lxc/backup/SERVIDOR-backup-2025-11-02.tar.xz' # ACOMPNAHAR CRESCIMENTO DO ARQUIVO
```
<BR>

Limpar o histórico do terminal de acesso (local ou remoto) encerrando a sessão atual:
```bash
history -c && history -w && > ~/.bash_history && unset HISTFILE && exit   # LIMPAR HISTORICO DE COMANDOS E SAIR
```


<BR><BR><BR><BR><BR>





<!-- # Versão aberta ao escrutínio público. -->
