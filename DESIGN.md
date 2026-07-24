# alert-triage — архитектура и требования (design stage)

Зафиксировано 2026-07-12 по обсуждению с владельцем; исходный разбор —
feedback-relay#17. Разработка — после пилота feedback-relay M2.

## 1. Задача и позиция

Homelab + пачка сервисов + один человек без on-call-ротации: alert fatigue
максимальна, времени на разбор минимум. 90% алертов — флапы и известные
грабли; их должен отсеивать агент с доступом к метрикам и истории, а не
человек в 2 часа ночи.

**Что строим:** триаж-слой поверх Alertmanager.
**Чего НЕ строим:** свой алертинг (дедуп/resolve/silences/inhibition —
Alertmanager), свою AIOps-платформу, автономного агента с руками в проде.

## 2. Архитектура

```
Prometheus → Alertmanager ──webhook_config──▶ адаптер (CF Worker или
                                              event_type=alert_v1 в воркере
                                              feedback-relay — решить в A1)
                                                      │ chat.postMessage + metadata
                                                      ▼
                                              Slack #alerts (карантин)
                                                      │ pull-свип
                                                      ▼
GitHub issue в репо проекта  ◀────────────  Claude Routine:
(+ ✅/🚫/❓-реакции, дедуп                    • диагностика через MCP
 по fingerprint/permalink)                    (PromQL «что было час назад»,
                                              активные алерты, история)
                                              • «инцидент или флап»
                                              • группировка связанных
```

Протокол триажа — реюз `feedback-relay/triage/ROUTINE.md`: та же машина
состояний в реакциях, тот же принцип «спорное — ❓ и сводка владельцу»,
формат issue адаптируется под алерты (fingerprint, ссылки на Grafana,
firing/resolved-таймлайн).

## 3. Ландшафт (проверено 2026-07: не велосипед ли?)

| Решение | Что даёт | Почему не берём как есть |
|---|---|---|
| PagerDuty AIOps, incident.io, Datadog Bits AI, BigPanda | AI-группировка, разбор инцидентов | enterprise SaaS, не self-hosted, не тот масштаб |
| [Keep](https://github.com/keephq/keep) | open-source alert management + AI-корреляция | риск «слишком платформа» для нескольких VM — проверить в A0 |
| [HolmesGPT](https://github.com/robusta-dev/holmesgpt) (Robusta) | open-source AI root-cause агент по алертам | K8s-центричен (у нас Proxmox+VM) — забрать идеи промптов диагностики, проверить в A0 |

**Ниша**: карантин-канал → LLM-триаж → issues в пер-проектные репо,
human-in-the-loop через реакции, personal/homelab-масштаб — готового нет;
рынок целит в enterprise on-call. Новизна проекта — воркфлоу-слой, не
AI-диагностика.

## 4. MCP-роли (важный разрез)

- **Диагностическая рука агента = MCP, берём готовое:** официальный
  [mcp-grafana](https://github.com/grafana/mcp-grafana) (дашборды,
  datasources, alerting) + коммьюнити Prometheus/Alertmanager MCP.
  Не строить своё.
- **Транспорт алертов ≠ MCP:** MCP — pull-протокол «агент спрашивает»,
  алерт — push-событие. Доставка — webhook-адаптер; Routine подхватывает
  pull-свипом по расписанию (та же механика, что свип фидбека).
- **Протокол триажа как typed-MCP-тулзы** — см. feedback-relay#16: если
  появится, обслужит оба потока (`feedback_v1` / `alert_v1`).

## 5. Ограничение среды

Homelab (Prometheus/Alertmanager/Grafana в приватной RFC1918-подсети,
условно `192.168.X.0/24`) за Tailscale:

- диагностический MCP достижим из **локальных** Cowork-сессий (Tailscale на
  хосте) сразу;
- из облачных CCR-сессий — только через funnel/публичный endpoint с auth.

→ где живёт триаж-Routine (локально vs облако) — решение этапа A1; влияет
на расписание и на то, что доступно агенту при разборе.

Публичный репо: адреса/имена хостов homelab в конфиги не коммитим — только
примеры с плейсхолдерами (та же дисциплина, что в feedback-relay).

## 6. Этапы

| Этап | Содержимое | Гейт |
|---|---|---|
| **A0** | Гейт запуска: данные пилота feedback-relay M2 (жизнеспособность протокола) + по полдня на Keep и HolmesGPT (что берём/не берём) | feedback-relay#2 закрыт |
| **A1** | Webhook-адаптер: Alertmanager → Slack #alerts с metadata `alert_v1`; решить «отдельный воркер vs event_type в feedback-relay»; где живёт Routine | A0 |
| **A2** | Триаж-Routine для алертов: адаптация ROUTINE.md (fingerprint-дедуп, firing/resolved, маппинг alert → целевой репо); пилот с черновиками | A1 |
| **A3** | Диагностический контур: mcp-grafana + Prometheus MCP в Routine; «что было час назад», история срабатываний | A2 |
| **A4** | Статья («AI-триаж инцидентов из Alertmanager, готовых MCP и 200 строк воркера») | A2–A3 |

## 7. Открытые вопросы (перенесены из feedback-relay#17)

- [ ] Keep / HolmesGPT: что берём, что нет (A0)
- [ ] Один канал `#alerts` или per-severity; маппинг alert → целевой репо
- [ ] Где живёт Routine: локальный Cowork (Tailscale-доступ к MCP) vs CCR+funnel
- [ ] Отдельный воркер vs `event_type: alert_v1` в воркере feedback-relay
- [ ] Recovery-семантика: resolved-алерт — реакция на исходное сообщение или отдельное сообщение в тред
- [ ] Аутентификация входящего webhook (#3): `webhook_config` Alertmanager постит server-to-server, без Origin — Origin-защита из feedback-relay §8 здесь неприменима. Публичный CF Worker без проверки источника принимает поддельный алерт от любого, кто узнал URL. Варианты: shared secret в URL/заголовке, HMAC-подпись payload, Basic Auth в `webhook_config`. Зафиксировать механизм в A1, до кода адаптера
- [ ] Публикация содержимого алертов (#5): payload несёт внутренности homelab (`instance` с IP:портом, хостнеймы, Grafana-URL) — что из этого может попасть в issue публичного репо. Варианты: редакция адаптером/Routine до создания issue; alert-issues только в приватный репо, в публичные — ссылка; явный public/private-флаг в маппинге alert → репо. Та же политика — для диагностических выводов A3. Решить в A2
- [ ] Групповые payload Alertmanager (#6): `webhook_config` шлёт группу (`groupKey`, `alerts[]` — у каждого свой fingerprint/status; повторная доставка каждые repeat_interval, смешанные firing/resolved). Гранулярность сообщение↔группа/алерт, черновик схемы `alert_v1.event_payload` (fingerprint(ы), status, groupKey, severity, ключ маппинга), семантика повторной доставки. Зафиксировать в A1, до кода адаптера
- [ ] Память «известный флап» (#7): реакции — per-message, а тот же fingerprint приходит новым сообщением каждый повтор; где живёт состояние «это флап» (🚫-история по fingerprint; закрытые issues как реестр; flap-ledger; TTL «не переспрашивать N дней») и путь эскалации «был флап — стал инцидент». Решить в A2 вместе с адаптацией ROUTINE.md
