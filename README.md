# CitizenTech - Idoso em casa com mais segurança
**Global Solution 2026/2 · FIAP · Mastering Relational and Non-Relational Database**

## Integrantes
| Ilda Ester Mussungayi Afonso | RM: 568233 | Turma: 2TDPSA 


## 1. Problema
Muitos idosos moram sozinhos ou passam o dia sem companhia. Eles esquecem remédios e podem demorar a receber ajuda em uma queda ou emergência. A solução CitizenTech usa um dispositivo (Arduino/ESP32) na casa do idoso, com **botão de emergência** e sensores de **abertura da caixa de remédios** e de **movimento**. Cada ocorrência é registrada como um evento, e o **cuidador** (familiar, profissional ou vizinho) precisa ser avisado conforme a gravidade.

- **Cidadão beneficiado:** idosos que vivem sozinhos e seus cuidadores.
- **ODS:** 3 (Saúde e bem-estar) e 10 (Redução das desigualdades).

Esta entrega é a **camada de processamento de dados em Oracle PL/SQL**: ela recebe os eventos do dispositivo, classifica a gravidade e monta a fila de notificações.

## 2. Modelo de dados
4 tabelas relacionadas, todas com 10 ou mais registros de teste (exceto medicamentos, que tem 12, e eventos, que tem 20).

```
CIT_CUIDADOR 1 ──< N CIT_IDOSO 1 ──< N CIT_MEDICAMENTO
                         │                    │
                         └──< N CIT_EVENTO >──┘ (medicamento é opcional)
```

| Tabela | Descrição | Registros |
|--------|-----------|-----------|
| `CIT_CUIDADOR` | Quem recebe os alertas (nome, telefone, e-mail, tipo) | 10 |
| `CIT_IDOSO` | Pessoa monitorada; tem um `codigo_dispositivo` único (o Arduino/ESP32 da casa) | 10 |
| `CIT_MEDICAMENTO` | Remédios e horários (HH:MM) de cada idoso | 12 |
| `CIT_EVENTO` | Ocorrências: `BOTAO_EMERGENCIA`, `MEDICAMENTO_TOMADO`, `MEDICAMENTO_NAO_TOMADO`, `SEM_MOVIMENTO`; status `PENDENTE`, `NOTIFICADO` ou `RESOLVIDO` | 20 |

O DDL inclui chaves primárias e estrangeiras, `CHECK` (tipo, status, formato de horário) e índices nas chaves estrangeiras.

## 3. Regras de negócio
1. **Classificação de gravidade** (decisão `IF/ELSIF/ELSE`), aplicada ao evento pendente mais antigo:

   | Tipo do evento | Gravidade | Ação recomendada |
   |----------------|-----------|------------------|
   | `BOTAO_EMERGENCIA` | CRÍTICA | Ligar para o cuidador e acionar o SAMU (192) |
   | `SEM_MOVIMENTO` | ALTA | Ligar para o idoso e avisar o cuidador |
   | `MEDICAMENTO_NAO_TOMADO` | MÉDIA | Enviar lembrete ao idoso e ao cuidador |
   | demais | BAIXA | Apenas registrar no histórico |

2. **Identificação pelo dispositivo:** o Arduino envia o `codigo_dispositivo`; o banco descobre de qual idoso se trata. Dispositivo desconhecido gera o erro `ORA-20001`.
3. **Fila de notificações:** todo evento `PENDENTE` entra na fila, do mais antigo para o mais recente, com o cuidador responsável e o telefone.
4. **Resumo por cuidador:** conta os eventos pendentes de cada idoso acompanhado.

## 4. O que cada script entrega
| Arquivo | Conteúdo |
|---------|----------|
| `01_ddl_tabelas.sql` | Criação das 4 tabelas, restrições e índices (apaga e recria, pode rodar várias vezes) |
| `02_carga_dados.sql` | Dados de teste fictícios e conferência de contagem |
| `03_plsql.sql` | Blocos anônimos, cursor explícito e 2 procedures |

Requisitos da disciplina atendidos em `03_plsql.sql`:

| Requisito | Onde |
|-----------|------|
| Bloco anônimo com variáveis, `SELECT INTO` e `DBMS_OUTPUT` | Bloco anônimo 1 |
| Decisão `IF/ELSIF/ELSE` | Classificação de gravidade (bloco 1) |
| Repetição (`FOR`) | Lista os idosos com eventos pendentes (bloco 1) e o loop de `sp_resumo_cuidador` |
| Cursor explícito (`OPEN/FETCH/CLOSE`) | Bloco anônimo 2: fila de notificações |
| Procedimento de operação | `sp_registrar_evento(codigo_dispositivo, tipo, id_medicamento, observacao)` |
| Procedimento de consulta/processamento | `sp_resumo_cuidador(id_cuidador, total_pendentes OUT)` |

## 5. Como executar
Pré-requisitos: Oracle (FIAP) e SQL Developer.

1. Conecte-se com o seu usuário (RM) e senha.
2. Execute **como script (F5)**, nesta ordem:
   1. `01_ddl_tabelas.sql`
   2. `02_carga_dados.sql` (o resultado final deve mostrar 10, 10, 12 e 20)
   3. `03_plsql.sql`
3. Abra a aba **Saída do Script** (e habilite `DBMS_OUTPUT` se necessário) para ver os resultados.

> O `03_plsql.sql` termina com o teste de erro `ESP32-9999`. A mensagem `ORA-20001: Dispositivo nao cadastrado` é **esperada**: demonstra o tratamento de exceção.

## 6. Resultados e evidências
Os prints estão na pasta `evidencias/`:

| Evidência | Arquivo |
|-----------|---------|
| Contagem das tabelas | `evidencias/01_contagem_tabelas.png` |
| Bloco anônimo (decisão e repetição) | `evidencias/02_bloco_anonimo.png` |
| Cursor: fila de notificações | `evidencias/03_cursor.png` |
| Procedures: resumo antes e depois do evento (total pendente do cuidador 3 vai de 0 para 1) | `evidencias/04_procedures.png` |
| Erro tratado (`ORA-20001`) | `evidencias/05_erro_tratado.png` |

## 7. Conexão com as outras disciplinas
O mesmo domínio (Cuidador → Idoso → Medicamento/Evento) é usado na API (.NET e Java), no app Flutter, no protótipo IoT e na arquitetura ArchiMate. Os eventos que o Arduino envia alimentam a tabela `CIT_EVENTO`.
