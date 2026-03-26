---
name: conventional-commits
description: >
  Padrão de mensagem para commits git. Use sempre que um commit for ser criado
  ou a mensagem de commit precisar ser escrita ou revisada. Carregue automaticamente
  ao realizar qualquer operação de git commit.
user-invocable: true
disable-model-invocation: false
---

## Prévia obrigatória antes de qualquer commit

Antes de executar o `git commit`, **sempre** exiba uma prévia para confirmação do usuário no seguinte formato:

```
Branch destino: <nome-do-branch>

Arquivos adicionados:
  + caminho/do/arquivo.ext
  (omitir seção se não houver)

Arquivos removidos:
  - caminho/do/arquivo.ext
  (omitir seção se não houver)

Arquivos alterados:
  ~ caminho/do/arquivo.ext
  (omitir seção se não houver)

Mensagem do commit:
  <tipo>(<escopo>): <descrição>

  <corpo, se houver>
```

Só prossiga com o commit após confirmação explícita do usuário. Se o usuário rejeitar ou sugerir alterações, ajuste e exiba a prévia novamente antes de tentar o commit.

---

Toda mensagem de commit deve seguir o padrão Conventional Commits.

---

## Formato

```
<tipo>(<escopo>): <descrição>

[corpo opcional]

[rodapé(s) opcional(is)]
```

- **tipo** e **descrição** são obrigatórios.
- **escopo** é opcional — use quando ajuda a localizar onde a mudança ocorre (ex: `auth`, `crawler`, `api`).
- Linha do título: máximo 72 caracteres.
- Descrição em **minúsculas**, sem ponto final, no **imperativo** ("adiciona" não "adicionado").

---

## Tipos

| Tipo | Quando usar |
|---|---|
| `feat` | Nova funcionalidade para o usuário |
| `fix` | Correção de bug |
| `refactor` | Mudança de código que não adiciona feature nem corrige bug |
| `test` | Adição ou correção de testes |
| `docs` | Apenas documentação |
| `chore` | Tarefas de manutenção, configs, dependências |
| `perf` | Melhoria de performance |
| `ci` | Mudanças em pipelines de CI/CD |
| `build` | Mudanças no sistema de build ou dependências externas |
| `revert` | Reverte um commit anterior |
| `style` | Formatação, ponto-e-vírgula, espaços — sem mudança de lógica |

---

## Breaking changes

Dois formatos aceitos:

**1. Ponto de exclamação após o tipo:**
```
feat!: remove suporte ao endpoint legado /v1/events
```

**2. Rodapé `BREAKING CHANGE`:**
```
feat(api): altera formato de resposta de odds

BREAKING CHANGE: campo `odd` renomeado para `oddDecimal` em todos os endpoints.
```

---

## Corpo e rodapé

- Separe título do corpo com **uma linha em branco**.
- O corpo explica o **porquê**, não o **o quê** (o diff já mostra o quê).
- Rodapés seguem o formato `Token: valor` ou `Token #referência`.

```
fix(crawler): corrige parsing de odds nulas na Betano

Odds nulas retornadas pela API causavam NullPointerException no mapper.
Adicionada verificação defensiva antes da conversão para Double.

Closes #42
```

---

## Exemplos válidos

```
feat(crawler): adiciona suporte ao site Hiperbet via Altenar
fix(service): evita divisão por zero no cálculo de margem
refactor(json): extrai AltenarJsonMapper para classe dedicada
test(crawler): adiciona fixture JSON real da Betano
chore: atualiza dependência do Playwright para 1.42
docs: documenta níveis de complexidade de crawlers
feat!: remove endpoint /api/v1/partidas descontinuado
```

## Exemplos inválidos

```
fix: Corrigido bug                 ← maiúscula e passado
feat: adiciona nova funcionalidade.  ← ponto final
update crawler                     ← sem tipo
feat: adiciona suporte ao site Hiperbet via Altenar e corrige bug de parsing de odds nulas no mapper JSON que causava NullPointerException  ← título longo demais
```
