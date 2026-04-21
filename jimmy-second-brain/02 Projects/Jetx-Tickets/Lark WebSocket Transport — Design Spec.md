# Lark WebSocket Transport — Design Spec

  

## 1. Context

  

The Lark ticketing bot currently exposes an HTTP `POST /lark/webhook` endpoint for both event subscriptions and card action callbacks. This requires a public HTTPS URL (ngrok in dev), an encrypt key, a verification token, and signature verification.

  

The user has elected to switch to Lark's long-connection / WebSocket delivery mode: the bot opens an outbound WS connection to Lark, no public URL, no signatures, no encryption. The switch is a **full commit** — the HTTP path is removed, not kept as fallback.

  

Through direct inspection of `lark-oapi==1.5.3`, we confirmed:

- `lark.ws.Client._handle_data_frame` dispatches `MessageType.EVENT` frames through `EventDispatcherHandler.do_without_validation` correctly.

- `lark.ws.Client._handle_data_frame` at lines 262–267 **silently drops `MessageType.CARD` frames** (card button clicks, form submissions). This is a gap in the stock SDK.

- The dispatcher itself supports card actions via `register_p2_card_action_trigger`; only the WS transport lacks the dispatch line.

- 1.5.3 is the latest available PyPI release; no upstream fix exists.

  

## 2. Goal

  

Replace HTTP webhook transport with Lark WebSocket transport, including full support for card action callbacks via a minimal `lark.ws.Client` subclass.

  

## 3. Architecture

  

Run a Lark WS client in a daemon thread, started from a FastAPI lifespan hook. Because the WS client's `start()` method blocks on its own internal asyncio loop, we isolate it from FastAPI's uvicorn loop. Because the SDK invokes handlers synchronously from inside that loop, we bridge to async business logic via a dedicated second loop running in its own thread.

  

## 4. Components

  

| File | Role | Change type |

|------|------|------------|

| `lark-tickets/app/lark/dispatch.py` | Action handlers (`_handle_*`), dispatch table, `dispatch_card_action`, `handle_message_event`, `_handle_ticket_command`, helpers. No HTTP router. | RENAME from `webhook.py` + remove router |

| `lark-tickets/app/lark/ws_client.py` | `_AsyncBridge`, `PatchedWSClient`, `_on_message`, `_on_card_action`, `start_ws_client`, `stop_ws_client`. | CREATE |

| `lark-tickets/app/main.py` | FastAPI lifespan wires WS client start/stop. No webhook router mount. | MODIFY |

| `lark-tickets/app/config.py` | `Settings` drops `lark_verification_token` and `lark_encrypt_key`. | MODIFY |

| `lark-tickets/.env.example` | Drops the two env vars. | MODIFY |

| `lark-tickets/tests/conftest.py` | Drops the two `monkeypatch.setenv` lines. | MODIFY |

| `lark-tickets/tests/test_config.py` | Any assertions about the two dropped fields are removed. | MODIFY |

| `lark-tickets/tests/test_dispatch.py` | Unit tests for `_toast`, handler return shape, `dispatch_card_action`, `handle_message_event`. | CREATE |

| `lark-tickets/tests/test_ws_client.py` | Unit tests for `_AsyncBridge`, `PatchedWSClient._handle_data_frame`, `_on_message`, `_on_card_action`, `start_ws_client`. | CREATE |

  

Untouched (out of scope): `app/lark/base.py`, `app/lark/cards.py`, `app/lark/notifications.py`, `app/lark/client.py`, `app/routers/admin.py`.

  

## 5. Data Flow

  

### Message events (user DMs the bot)

```

Lark server

  └─> WS frame (MessageType.EVENT)

      └─> PatchedWSClient._handle_data_frame

          └─> EventDispatcherHandler.do_without_validation

              └─> _on_message (sync, registered via register_p2_im_message_receive_v1)

                  └─> _bridge.run(handle_message_event(...))

                      └─> _handle_ticket_command  (if the text is "/ticket")

                          └─> send_card_dm(...)

```

  

### Card actions (user clicks a button / submits a form)

```

Lark server

  └─> WS frame (MessageType.CARD)   <-- stock SDK drops this; patched subclass dispatches

      └─> PatchedWSClient._handle_data_frame

          └─> EventDispatcherHandler.do_without_validation

              └─> _on_card_action (sync, registered via register_p2_card_action_trigger)

                  └─> _bridge.run(dispatch_card_action(action_value, form_value, operator_id))

                      └─> _handle_<action>(merged_value, operator_id)

                          └─> returns {"toast": {...}}

              returns P2CardActionTriggerResponse(toast=...)

      <- Response serialized onto the same WS frame (confirms in the Lark UI as toast)

```

  

## 6. Error Handling

  

- Handler exception → caught by `dispatch_card_action` → `{"toast": {"type": "error", "content": "Something went wrong. Please try again."}}`.

- Unknown action name → `{"toast": {"type": "error", "content": "Unknown action"}}`.

- Dispatcher exception inside `_handle_data_frame` → caught, sends `Response(code=500)` on the WS frame.

- DM send failures → already logged-not-raised by `notifications.py`.

- WS connection drop → `lark.ws.Client` has `auto_reconnect=True` by default; bridge loop outlives reconnects.

  

## 7. Definition of Done

  

- `pytest` green across `tests/test_base.py`, `tests/test_client.py`, `tests/test_config.py`, `tests/test_dispatch.py` (new), `tests/test_ws_client.py` (new).

- `from app.main import app` loads without error when env is populated (dummy values acceptable).

- No E2E verification against real Lark is required by this plan. That happens separately after merge.

  

## 8. Risks

  

| Risk | Mitigation |

|------|------------|

| SDK-internal surface (`_get_by_key`, `_combine`, `_write_message`, `_event_handler`, `MessageType`, `Response`, `HEADER_*`) changes on `lark-oapi` bump | Pin `lark-oapi==1.5.3` in `requirements.txt`; NOTE comment in `ws_client.py` flags re-verification needed on bump. |

| Blocking bridge call deadlocks | Dedicated bridge loop is isolated from SDK's loop; `asyncio.run_coroutine_threadsafe(...).result(timeout=60)` caps any single call. |

| No clean shutdown API on `lark.ws.Client` | Daemon thread; dies with process on SIGTERM. Bridge loop stops cleanly on FastAPI shutdown. |