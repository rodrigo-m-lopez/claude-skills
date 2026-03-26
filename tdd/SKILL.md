---
name: tdd
description: >
  Processo TDD (Test-Driven Development). Use quando o usuário pedir para implementar
  uma nova funcionalidade, corrigir um bug, ou sempre que código de produção precisar
  ser escrito ou modificado. Carregue automaticamente ao iniciar qualquer tarefa de
  implementação ou correção de código.
user-invocable: true
disable-model-invocation: false
---

Todo código de produção — nova funcionalidade ou correção de bug — deve seguir este processo. Nunca escreva código de produção antes de um teste correspondente existir e falhar.

---

## Nova funcionalidade

1. **Escreva o(s) teste(s) primeiro.**
   - O teste deve descrever o comportamento esperado e **falhar**.
   - Use `@Test` com `assertThat`, `assertEquals`, etc.
   - Não escreva código de produção ainda.

2. **Confirme que o teste falha pelo motivo correto.**
   - Deve falhar por comportamento ausente, não por erro de compilação ou setup mal configurado.
   - Execute apenas o teste novo para diagnóstico rápido.

3. **Implemente o código mínimo** para fazer o teste passar.
   - Sem over-engineering: apenas o suficiente para verde.

4. **Confirme que todos os testes passam** (regressão).

5. **Refatore** com confiança, mantendo os testes verdes.

---

## Correção de bug

1. **Reproduza o bug com um teste.**
   - Crie um teste que demonstre o comportamento errado atual — ele deve **falhar**.
   - Um teste que já passa antes de qualquer correção não é útil — revise-o.

2. **Confirme que o teste falha pelo motivo correto.**

3. **Corrija o código** para fazer o teste passar.

4. **Confirme que todos os testes passam** (regressão).

---

## Regras invariáveis

- Um teste que passa antes de qualquer implementação é inútil — revise-o.
- Código de produção só existe se houver teste correspondente que o exigiu.
- Testes não devem testar implementação; devem testar **comportamento observável**.

---

## Tipos de teste

| Tipo | Escopo | Característica |
|---|---|---|
| **Unidade** | Classe/função isolada | Sem I/O, sem framework, usa builders/factories para montar estado |
| **Integração** | Múltiplos componentes + framework | Sobe contexto real (ex: `@SpringBootTest`), moca dependências externas (ex: `@MockBean`) |
| **E2E** | Sistema completo | Chama dependências reais; não deve rodar em CI por padrão |

---

## Comandos de referência

Adapte conforme o build tool do projeto:

```bash
# Rodar apenas um teste (diagnóstico rápido)
./mvnw test -Dtest=NomeDaClasseTest          # Maven
./gradlew test --tests NomeDaClasseTest      # Gradle

# Rodar todos os testes (regressão)
./mvnw test
./gradlew test
```
