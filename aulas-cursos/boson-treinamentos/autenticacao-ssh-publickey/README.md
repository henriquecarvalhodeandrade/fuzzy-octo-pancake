# Configuração de Conexão SSH por Par de Chaves

**Guia Prático e Detalhado de Autenticação Remota Segura em Linux**

> Autor do vídeo: Henrique
> Vídeo de referência: [YouTube (XanFMc-tQg0)](https://www.youtube.com/watch?v=XanFMc-tQg0)
> Ambiente: Linux (Cliente / Servidor)

---

## 📋 Índice

1. [Introdução e Contextualização](#1-introdução-e-contextualização)
2. [Como Funciona a Autenticação por Chave SSH](#2-como-funciona-a-autenticação-por-chave-ssh)
3. [Preparação dos Ambientes](#3-preparação-dos-ambientes)
4. [Inspeção Inicial com Modo Verboso (-v)](#4-inspeção-inicial-com-modo-verboso--v)
5. [Passo a Passo Manual: Geração e Envio de Chaves](#5-passo-a-passo-manual-geração-e-envio-de-chaves)
6. [Método Automatizado: ssh-copy-id](#6-método-automatizado-usando-ssh-copy-id)
7. [Automatização de Passphrase com SSH Agent](#7-automatização-de-passphrase-com-ssh-agent)
8. [Teste de Conexão Final](#8-teste-de-conexão-final)

---

## 1. Introdução e Contextualização

Neste guia técnico detalhado, abordaremos o processo de transição do método tradicional de autenticação SSH por **senha** para o método seguro de autenticação por **par de chaves criptográficas** (Pública/Privada).

### Por que a autenticação por senha não é ideal?

A autenticação por senha expõe o servidor a ataques de força bruta ou interceptações. Se um invasor descobrir a senha do usuário remoto (ex: `vboxuser`), ele obtém acesso total à máquina servidor a partir de qualquer computador da rede.

---

## 2. Como Funciona a Autenticação por Chave SSH

O fluxo de autenticação por chaves assimétricas segue quatro etapas fundamentais:

1. **Solicitação de Conexão** — O computador Cliente solicita a conexão com o Servidor especificando a conta de usuário desejada (ex: `vboxuser@192.168.15.68`).
2. **Desafio de Identificação** — O servidor solicita a confirmação da identidade do cliente.
3. **Geração do Autenticador** — O cliente gera um autenticador assinado com sua **Chave Privada** (armazenada de forma estritamente confidencial no cliente) e o envia ao servidor.
4. **Validação e Descriptografia** — O servidor utiliza a **Chave Pública** do cliente (previamente autorizada no arquivo `authorized_keys`) para descriptografar e validar o autenticador, liberando o acesso sem trafegar senhas.

---

## 3. Preparação dos Ambientes

Nos testes realizados, foram utilizados dois hosts/máquinas virtuais:

| Função | Nome do Host / Usuário | Endereço IP |
|---|---|---|
| Servidor (Esquerda) | `vboxuser` | `192.168.15.68` |
| Cliente (Direita) | `Henrique` | IP Local do Cliente |

---

## 4. Inspeção Inicial com Modo Verboso (-v)

Para analisar os métodos de autenticação aceitos pelo servidor antes de criar as chaves, executa-se o SSH no modo verboso:

```bash
ssh -vl vboxuser 192.168.15.68
```

> **Observação:** o argumento `-v` ativa o log detalhado e `-l` especifica o nome de usuário do servidor.

**Resultado da Inspeção:** ao analisar os logs do terminal, observa-se a linha informando os métodos de autenticação permitidos: `publickey,password`. Como a chave pública ainda não está instalada no servidor, o SSH falha na tentativa de chave e faz o *fallback* para a autenticação por senha.

---

## 5. Passo a Passo Manual: Geração e Envio de Chaves

### Passo 1: Gerar o Par de Chaves Criptográficas (no Cliente)

No terminal do computador cliente, execute o comando de geração especificando o algoritmo RSA (o algoritmo legado DSA não é mais suportado por padrão em distribuições Linux modernas):

```bash
ssh-keygen -t rsa
```

- **Caminho do Arquivo:** pressione Enter para aceitar o local padrão (`~/.ssh/id_rsa`).
- **Passphrase (Senha da Chave):** digite uma frase secreta (no vídeo utilizada a palavra de teste `batata`). Em ambientes reais de produção, recomenda-se criar uma passphrase complexa com mais de 10 caracteres contendo letras maiúsculas, minúsculas, números e símbolos.

### Passo 2: Inspecionar o Diretório de Chaves

Acesse o diretório oculto `.ssh` no cliente para verificar os arquivos criados:

```bash
cd ~/.ssh
ls -la
```

- `id_rsa` — Chave Privada (**nunca** deve ser compartilhada).
- `id_rsa.pub` — Chave Pública (pode ser enviada ao servidor).

### Passo 3: Transferir a Chave Pública para o Servidor via SCP

Envie a chave pública para o diretório raiz (Home) do servidor através do protocolo SCP:

```bash
scp id_rsa.pub vboxuser@192.168.15.68:~/
```

### Passo 4: Configurar a Chave Autorizada no Servidor

No computador Servidor, acesse a máquina e adicione o conteúdo da chave pública dentro do arquivo de chaves autorizadas localizado na pasta `.ssh`:

```bash
# Adiciona o conteúdo do arquivo id_rsa.pub no final do arquivo authorized_keys
cat id_rsa.pub >> ~/.ssh/authorized_keys

# Limpa o arquivo id_rsa.pub temporário da Home
rm id_rsa.pub
```

Para conferir a gravação correta da chave pública:

```bash
cat ~/.ssh/authorized_keys
```

---

## 6. Método Automatizado: Usando ssh-copy-id

Todo o processo de transferência e adição da chave pública no arquivo `authorized_keys` do servidor (Passos 3 e 4) pode ser realizado com um único comando no cliente:

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub vboxuser@192.168.15.68
```

> **Vantagem do `ssh-copy-id`:** além de automatizar o envio, ele cria o diretório `.ssh` e o arquivo `authorized_keys` no servidor com as permissões POSIX corretas (ex: `700` para pasta e `600` para arquivo), prevenindo erros de permissão negada.

---

## 7. Automatização de Passphrase com SSH Agent

Para evitar a digitação repetitiva da passphrase a cada nova conexão SSH durante a sessão do sistema, utiliza-se o agente do SSH em segundo plano:

### 1. Iniciar o SSH Agent no Terminal Cliente

```bash
eval $(ssh-agent -s)
```

> No vídeo: `ssh-agent $SHELL` ou `ssh-agent -s`

### 2. Adicionar a Chave Privada ao Agente

```bash
ssh-add ~/.ssh/id_rsa
```

Digite a passphrase da chave (ex: `batata`) uma única vez. A partir deste momento, a chave fica guardada na memória do cliente até o encerramento da sessão ou reinicialização.

### 3. Comandos Úteis do SSH Agent

| Ação | Comando |
|---|---|
| Listar chaves salvas no agente | `ssh-add -l` |
| Remover chaves salvas do agente | `ssh-add -d` |

---

## 8. Teste de Conexão Final

Com a chave instalada no servidor e o agente ativo no cliente, a conexão SSH é realizada com sucesso de forma transparente e direta:

```bash
ssh vboxuser@192.168.15.68
```

A conexão é estabelecida imediatamente **sem solicitar a senha** do usuário remoto e **nem a passphrase** da chave, combinando segurança máxima com facilidade de uso.

---

## Link para video demosntração:
- https://www.youtube.com/watch?v=XanFMc-tQg0