# alert-triage

AI-триаж алертов поверх зрелого alerting-стека: Alertmanager шлёт webhook →
тонкий адаптер кладёт алерт в Slack-карантин → Claude Routine разбирает
(диагностика через Prometheus/Grafana MCP, «инцидент или флап») → аутком:
GitHub issue в репо нужного проекта или честное 🚫 «флап».

```
Alertmanager (webhook_config) → адаптер → Slack #alerts (metadata alert_v1)
                                                │
              GitHub issue в нужный репо  ◀── Claude Routine: диагностика,
              (+ ✅/🚫/❓-протокол)              группировка, human-in-the-loop
```

**Ключевая позиция:** не заменяем Alertmanager (дедуп, resolve, silences,
inhibition остаются у него) и не строим ещё одну AIOps-платформу. Новый
слой — только триаж-воркфлоу; агент ничего не может сломать: он читает,
предлагает и ставит реакции, право на действие остаётся у человека.

Протокол триажа переиспользуется из [feedback-relay](https://github.com/ignatenkofi/feedback-relay)
(карантин-канал, машина состояний в реакциях, дедуп, issue-формат) — фидбек
и алерты различаются только `event_type` в metadata.

## Статус

**Design stage.** Разработка стартует после пилота триажа фидбека
(feedback-relay [M2](https://github.com/ignatenkofi/feedback-relay/issues/2) —
колбэк-анкор: [feedback-relay#17](https://github.com/ignatenkofi/feedback-relay/issues/17)):
пилот покажет жизнеспособность триаж-протокола до того, как вешать на него
более требовательный поток.

Архитектура, сравнение с существующими решениями (Keep, HolmesGPT,
enterprise AIOps), разрез MCP-ролей и открытые вопросы — в [DESIGN.md](DESIGN.md).
Роадмап — в issues.
