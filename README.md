# rewriting-roteiro-shrek

Automação que mantém commits diários nas horas, reescrevendo o roteiro de **Shrek** palavra por palavra.

## O que é

Um "watchdog" de commits: ele garante que o repositório receba commits de forma contínua e imprevisível, sem nunca deixar passar mais de 20 horas entre um commit e outro. A cada commit, uma palavra do roteiro de Shrek é anexada a `roteiro.md`, montando o texto aos poucos.

## Como funciona

O agendamento é **determinístico e aleatório ao mesmo tempo**:

- Duas fontes rodam o mesmo binário (`scripts/daily_commit.go`): o **GitHub Actions** e um **cron local** da máquina.
- Ambas usam o **hash do último commit** como semente para sortear o próximo horário:
  - dias normais: de **240 a 480 minutos** (~4 commits/dia);
  - sextas, sábados e feriados (boost): de **120 a 360 minutos**;
  - limite de segurança: nunca mais que **20 horas** sem commit;
  - teto de **10 commits/dia**.
- Como a semente é a mesma (mesmo commit), as duas fontes calculam o mesmo próximo horário e se coordenam — quem rodar primeiro cria o commit, a outra vê o commit novo e respeita o novo sorteio.

Cada commit:

1. anexa **1 palavra** de `meu roteiro.txt` a `roteiro.md`;
2. atualiza `.daily-commit/word-state.json` (próximo índice) e `heartbeat.json`;
3. cria o commit e faz push para `main`.

Quando o texto chega ao fim, ele reinicia da primeira palavra (contando as voltas em `completed_runs`).

## Como roda

### Opção A — GitHub Actions (recomendado, serve para sempre)

O workflow `.github/workflows/daily-commit.yml` roda a cada 15 minutos na nuvem do GitHub. Vantagem: funciona mesmo com a máquina desligada.

### Opção B — Cron local

```cron
*/15 * * * * cd /home/user/script-commit/rewriting-roteiro-shrek && git fetch origin && git reset --hard origin/main && RANDOM_MIN_MINUTES_BETWEEN_COMMITS=240 RANDOM_MAX_MINUTES_BETWEEN_COMMITS=480 BOOST_RANDOM_MIN_MINUTES_BETWEEN_COMMITS=120 BOOST_RANDOM_MAX_MINUTES_BETWEEN_COMMITS=360 BOOST_FIXED_DATES=01-01,04-21,05-01,09-07,10-12,11-02,11-15,11-20,12-25 BOOST_DATES= SAFETY_MAX_MINUTES_WITHOUT_COMMIT=1200 MAX_COMMITS_PER_DAY=10 COMMIT_DAY_TIMEZONE=America/Sao_Paulo HEARTBEAT_FILE=.daily-commit/heartbeat.json SOURCE_TEXT_FILE="meu roteiro.txt" OUTPUT_TEXT_FILE=roteiro.md WORD_STATE_FILE=.daily-commit/word-state.json TARGET_BRANCH=main FORCE_COMMIT=false SKIP_PUSH=false /usr/bin/go run ./scripts/daily_commit.go >> $HOME/daily-commit.log 2>&1
```

O `git fetch origin && git reset --hard origin/main` antes de rodar mantém a máquina alinhada com o GitHub — sem isso as duas fontes brigam e o push local é rejeitado.

## Por que a atividade fica verde

O gráfico de contribuições do GitHub só conta commits em que o **e-mail do autor** pertence a uma conta verificada. Por isso o workflow configura a identidade:

```yaml
git config user.name "user.name"
git config user.email "email@gmail.com"
```

Authorando com o e-mail da conta (`email@email.com`), tanto os commits do Actions quanto os do cron contam como atividade do autor.

## Arquivos

| Arquivo | Papel |
|---|---|
| `scripts/daily_commit.go` | o watchdog em si (agenda, sorteio, palavra, push) |
| `meu roteiro.txt` | texto-fonte (não versionado como saída) |
| `roteiro.md` | saída gerada — 1 palavra por commit |
| `.daily-commit/heartbeat.json` | último horário/estado do watchdog |
| `.daily-commit/word-state.json` | índice da próxima palavra |
| `.github/workflows/daily-commit.yml` | cron do GitHub Actions |

## Rodar manualmente

```bash
# força um commit agora (sem esperar o sorteio)
FORCE_COMMIT=true go run ./scripts/daily_commit.go

# cria commit local sem push (teste)
SKIP_PUSH=true go run ./scripts/daily_commit.go
```

## Troublehooting rápido

| Sintoma | Solução |
|---|---|
| `non-fast-forward` / push rejeitado | máquina está atrás do remoto → `git fetch origin && git reset --hard origin/main` (o cron já faz isso sozinho agora) |
| commit não aparece no gráfico | confira se o autor é `user <email.@gmail.com>` |
| texto reiniciando | normal — o roteiro chegou ao fim e recomeçou (veja `completed_runs` no word-state) |
