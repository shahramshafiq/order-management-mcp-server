<div align="center">

# Order Management MCP Server

An MCP server with six inventory and order tools, plus a guardrailed OpenAI chat assistant that uses them.

![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.142-009688?style=flat-square&logo=fastapi&logoColor=white)
![MCP](https://img.shields.io/badge/MCP-stdio-5A4FCF?style=flat-square)
![OpenAI](https://img.shields.io/badge/OpenAI-API-412991?style=flat-square)
![NeMo Guardrails](https://img.shields.io/badge/NeMo%20Guardrails-0.24-76B900?style=flat-square&logo=nvidia&logoColor=white)

</div>

## Overview

Small stores track stock, reservations and order status in several places, and it is easy for the numbers to drift apart. This project keeps them consistent in one MCP server that exposes six tools: add a product, search the catalog, create an order, move an order through its statuses, receive a supplier restock, and generate an inventory report. A FastAPI app launches the server as a subprocess and lets an OpenAI model call those tools from a chat page. Every user message and every assistant reply goes through NeMo Guardrails checks first, so jailbreak and prompt-injection attempts are blocked before they reach the assistant or the user.

### Architecture

```text
Browser chat page (frontend/)
        |
        v
FastAPI  POST /chat
        |
        +-- 1. input check:  NeMo Guardrails (self check input)
        |
        +-- 2. assistant loop: OpenAI chat completions, up to 4 tool rounds
        |          |
        |          v
        |      MCP client session (stdio) to the MCP server (6 tools)
        |                                        |
        |                                        v
        |                                   JSON files in data/
        |
        +-- 3. output check: NeMo Guardrails (self check output)
        |
        v
JSON reply: answer and the names of the tools used
```

The FastAPI process starts `app/mcp_server/server.py` over the `stdio` transport at startup and keeps one client session open for the whole lifetime of the app. All data lives in plain JSON files under `data/`, with no database.

### Stock model

- `stock_quantity` is the number of physical units on the shelf.
- `reserved_quantity` is the number of units promised to a pending order that has not shipped.
- `available_quantity` is `stock_quantity - reserved_quantity`, and it is the number a new order is checked against.

Creating an order reserves stock. Confirming it turns the reservation into a sale (`stock_quantity` drops and the reservation clears). Cancelling a pending order releases the reservation, and cancelling a confirmed order that has not shipped reverses the sale. A shipped order cannot be cancelled.

## Key Features

- **Six MCP tools over stdio:** `add_product`, `search_inventory`, `create_order`, `update_order_status`, `restock_product` and `get_inventory_report`, all registered in `app/mcp_server/server.py`.
- **Reserved versus available stock:** orders are checked against available units, and stock status (`in_stock`, `low_stock`, `out_of_stock`) is derived from available units and the product's reorder threshold.
- **Validation-first order creation:** every line item is checked before any stock is reserved, so one bad item changes nothing. Duplicate product lines are combined, the unit price is snapshotted onto the order, and money is handled as `Decimal` with discount applied before tax.
- **Order status state machine:** only the allowed transitions are accepted, each with its matching stock effect, so repeating a transition cannot deduct stock twice.
- **Inventory movement log:** every reservation, release, sale, sale reversal and restock is recorded in `data/movements.json` with the order number or purchase order reference.
- **Restocks against purchase orders:** a purchase order reference can only be received once, and the supplier must match the product's assigned supplier.
- **Inventory report:** current inventory levels and value (based on the latest restock unit cost), low-stock and out-of-stock lists, order counts and revenue for a date range, the top five best sellers, and total stock received.
- **Input and output guardrails:** NeMo Guardrails runs an LLM-judged check on every message and every reply, using prompts written for this domain (`guardrails_config/prompts.yml`), including attempts to switch off business rules such as the stock check.

## Tech Stack

| Area | Technology |
|---|---|
| Language | Python 3.12 (backend), JavaScript, HTML and CSS (frontend) |
| Web framework | FastAPI, Uvicorn |
| Protocol | Model Context Protocol Python SDK (`mcp` 2.2.0), stdio transport |
| LLM | OpenAI Chat Completions with tool calling (model set by `OPENAI_MODEL`) |
| Safety | NeMo Guardrails 0.24.1 (input and output self-check rails) |
| Data and validation | Pydantic 2, pydantic-settings, JSON file storage |
| Testing | pytest |

## Project Structure

```text
app/
  main.py                 FastAPI app, starts the MCP server as a stdio subprocess
  config.py               Settings loaded from environment variables
  routes/chat.py          POST /chat with input and output guardrail checks
  services/               Assistant tool-calling loop, MCP client, guardrails, memory, prompts
  mcp_server/
    server.py             Registers the six MCP tools
    models.py             Pydantic models and enums for products, orders and movements
    storage.py            JSON file read and write helpers
    tools/                One module per tool
  utils/                  Labeled error classes and logging setup
data/                     JSON files for products, orders, suppliers and movements
guardrails_config/        NeMo Guardrails config, prompts and rails
frontend/                 Plain HTML, CSS and JavaScript chat page
tests/                    pytest suite (24 tests)
```

## Getting Started

### Prerequisites

- Windows with Python 3.12. The app currently launches the MCP server with `venv/Scripts/python.exe`, so the virtual environment must be named `venv` and sit in the project root.
- An OpenAI API key.
- Node.js, only if you want to try the MCP server on its own with MCP Inspector.

### Installation

```powershell
py -3.12 -m venv venv
venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
```

Then open `.env` and fill in the values. The app reads these environment variables:

| Variable | Purpose |
|---|---|
| `OPENAI_API_KEY` | API key for the assistant and the guardrail checks |
| `OPENAI_MODEL` | Model used by the assistant |
| `OPENAI_BASE_URL` | API base URL (`https://api.openai.com/v1` in the example file) |
| `GUARDRAILS_MODEL` | Model used by the NeMo Guardrails checks |
| `CURRENCY` | Currency label inserted into the assistant's system prompt |

Optional overrides with defaults: `GUARDRAILS_CONFIG_PATH`, `PRODUCTS_FILE`, `ORDERS_FILE`, `SUPPLIERS_FILE`, `MOVEMENTS_FILE` and `IDEMPOTENCY_FILE`.

### Run

Start the chat app from the project root:

```powershell
uvicorn app.main:app --port 8000
```

Open `http://localhost:8000` for the chat page. A health check is available at `http://localhost:8000/health`.

To try the six tools directly, without FastAPI or OpenAI, use MCP Inspector:

```powershell
npx @modelcontextprotocol/inspector venv/Scripts/python.exe -m app.mcp_server.server
```

## Usage

Example messages for the chat page:

- "add 20 wireless mice at 2500 each"
- "what's low on stock?"

Or call the API directly:

```bash
curl -X POST http://localhost:8000/chat -H "Content-Type: application/json" -d '{"message": "What is low on stock?"}'
```

A successful reply looks like `{"message": "...", "answer": "...", "tool_calls": ["search_inventory"]}`. Blocked or failed requests return `{"error": "CODE", "message": "..."}` with the codes `BLOCKED_INPUT`, `BLOCKED_OUTPUT`, `ASSISTANT_ERROR` or `INTERNAL_SERVER_ERROR`.

### MCP tools

| Tool | What it does |
|---|---|
| `add_product` | Registers a product with a unique SKU and an existing supplier. Prices are sent as strings and parsed as `Decimal`. |
| `search_inventory` | Searches name or SKU, filters by category, supplier and stock status, sorts by name, price or available quantity, and paginates. |
| `create_order` | Reserves stock for one or more items and starts the order as `pending`. Accepts `discount_percent` and `tax_percent` as strings. |
| `update_order_status` | Moves an order to a new status, using the order's internal `id` rather than its `order_number`. |
| `restock_product` | Receives stock for a SKU against a purchase order reference and unit cost. Never touches reservations. |
| `get_inventory_report` | Reports inventory and activity for `start_date` (inclusive) to `end_date` (exclusive), optionally scoped by category or supplier. |

Example `add_product` arguments:

```json
{"name": "Wireless Mouse", "sku": "MOU-001", "category": "Electronics", "unit_price": "2500.00", "initial_stock": 20, "reorder_threshold": 5, "supplier_id": "SUP-001"}
```

Allowed order status transitions: pending to confirmed, pending to cancelled, confirmed to shipped, confirmed to cancelled, and shipped to delivered. Anything else is rejected as `INVALID_TRANSITION`.

### Error codes

Tools raise labeled exceptions defined in `app/utils/errors.py`:

| Code | Raised when |
|---|---|
| `NOT_FOUND` | A referenced product, order or supplier does not exist |
| `DUPLICATE` | A SKU or purchase order reference is already in use |
| `VALIDATION_ERROR` | Input is invalid, for example a negative quantity, a malformed date or a supplier that does not match the product |
| `INVALID_TRANSITION` | The order status change is not allowed from its current state |
| `INSUFFICIENT_STOCK` | An order asks for more than the available quantity |

## Testing

```powershell
pytest tests/ -v
```

The suite has 24 tests across six files, one per tool. They call the tool functions directly and make no API calls. `tests/conftest.py` points every test at fresh temporary copies of the data files, so the files in `data/` are never touched. Settings load when `app/config.py` is imported, so a `.env` file must exist before running the tests (copying `.env.example` is enough).

## Limitations and Roadmap

- The MCP server is launched with the hard-coded path `venv/Scripts/python.exe`, so the app currently runs on Windows only. Using the current interpreter instead would remove that restriction.
- Chat history is kept in one in-memory list shared by all clients, and it resets when the app restarts.
- Storage is plain JSON files with a write lock inside one process. It suits a demo or a single user, not concurrent writers, and a database would be the next step.
- Helper functions for idempotency keys exist in `app/mcp_server/storage.py`, but no tool uses them yet.

## Author

**Shahram Shafiq** - [GitHub](https://github.com/shahramshafiq) | [LinkedIn](https://www.linkedin.com/in/shahramshafiq/)
