---
name: computer-use-skills
description: Use when working with computer use or browser automation on Vertex AI using google-genai Python SDK — imports ComputerUse, or requests involving "computer use", "browser automation", "web automation", "computer control", "GUI automation", "click", "type text", "navigate browser". Korean: "컴퓨터 사용", "브라우저 자동화", "웹 자동화", "화면 제어", "GUI 자동화".
version: 1.0.0
---

# Computer Use Skills — google-genai Python SDK Reference

> Reference: [Computer use](https://cloud.google.com/vertex-ai/generative-ai/docs/computer-use)

---

## Important: Preview Feature

> **Computer Use is currently a Preview feature.**
> - Only supported on `gemini-3-flash` (Preview model)
> - Not recommended for production workloads
> - Requires explicit safety acknowledgement on every request

---

## Environment Setup

### Install Libraries

```bash
pip install --upgrade google-genai
pip install playwright  # for browser automation
playwright install chromium
```

### Required Environment Variables

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
export GOOGLE_GENAI_USE_VERTEXAI="True"
```

### Client Initialization

```python
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(http_options=HttpOptions(api_version="v1"))
```

---

## § 1. Basic Computer Use Pattern

### When: automating browser tasks via Gemini screenshot + action loop

Computer Use works as a multi-turn loop:
1. Send task + screenshot to Gemini
2. Gemini returns actions to perform
3. Execute actions in browser
4. Capture new screenshot
5. Repeat until task is complete

```python
from google import genai
from google.genai.types import (
    ComputerUse,
    Environment,
    GenerateContentConfig,
    HttpOptions,
    Part,
    Tool,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

def send_to_gemini(screenshot_bytes: bytes, task: str, history: list) -> object:
    """Send screenshot and task to Gemini, return the response."""
    contents = history + [
        {
            "role": "user",
            "parts": [
                Part.from_text(task) if not history else Part.from_text("Current state:"),
                Part.from_bytes(data=screenshot_bytes, mime_type="image/png"),
            ],
        }
    ]

    response = client.models.generate_content(
        model="gemini-3-flash",   # only model that supports computer use
        contents=contents,
        config=GenerateContentConfig(
            tools=[
                Tool(
                    computer_use=ComputerUse(
                        environment=Environment.ENVIRONMENT_BROWSER,
                    )
                )
            ],
            safety_acknowledgement=True,   # required — must be True
        ),
    )

    return response
```

> **Rule:**
> - `safety_acknowledgement=True` is **mandatory**. Requests fail without it.
> - Only `gemini-3-flash` supports Computer Use.
> - Always implement human confirmation before executing actions.

---

## § 2. Action Types

### When: interpreting and executing actions returned by Gemini

Gemini returns `function_call` parts with one of these action types:

| Action | Parameters | Description |
|--------|-----------|-------------|
| `click_at` | `coordinate`, `button` | Click at screen coordinates |
| `type_text_at` | `coordinate`, `text` | Type text at coordinates |
| `navigate` | `url` | Navigate browser to URL |
| `scroll` | `coordinate`, `direction`, `amount` | Scroll at coordinates |
| `key_combination` | `keys` | Press keyboard shortcut |
| `hover_at` | `coordinate` | Move mouse to coordinates |
| `drag_and_drop` | `startCoordinate`, `endCoordinate` | Drag from start to end |
| `screenshot` | — | Request a new screenshot |
| `wait` | `duration_ms` | Wait for specified milliseconds |

### Coordinate System

> **Important:** Computer Use uses a **1000×1000 normalized coordinate grid**, regardless of actual screen resolution.
> - `[0, 0]` = top-left corner
> - `[1000, 1000]` = bottom-right corner
> - Scale coordinates to actual screen size before executing

```python
def scale_coordinate(coord: list, screen_width: int, screen_height: int) -> tuple:
    """Scale normalized 1000x1000 coordinates to actual screen pixels."""
    x = int(coord[0] * screen_width / 1000)
    y = int(coord[1] * screen_height / 1000)
    return x, y
```

---

## § 3. Full Browser Automation Loop

### When: running a complete multi-turn browser automation task

```python
import time
from playwright.sync_api import sync_playwright
from google import genai
from google.genai.types import (
    ComputerUse,
    Environment,
    GenerateContentConfig,
    HttpOptions,
    Part,
    Tool,
)

client = genai.Client(http_options=HttpOptions(api_version="v1"))

SCREEN_WIDTH = 1280
SCREEN_HEIGHT = 800

def take_screenshot(page) -> bytes:
    return page.screenshot()

def execute_action(page, action_name: str, action_args: dict) -> bool:
    """Execute a single action. Returns True if done."""
    if action_name == "navigate":
        page.goto(action_args["url"])

    elif action_name == "click_at":
        x, y = scale_coordinate(action_args["coordinate"], SCREEN_WIDTH, SCREEN_HEIGHT)
        button = action_args.get("button", "left")
        page.mouse.click(x, y, button=button)

    elif action_name == "type_text_at":
        x, y = scale_coordinate(action_args["coordinate"], SCREEN_WIDTH, SCREEN_HEIGHT)
        page.mouse.click(x, y)
        page.keyboard.type(action_args["text"])

    elif action_name == "scroll":
        x, y = scale_coordinate(action_args["coordinate"], SCREEN_WIDTH, SCREEN_HEIGHT)
        direction = action_args.get("direction", "down")
        amount = action_args.get("amount", 3)
        delta = amount * 100 if direction == "down" else -amount * 100
        page.mouse.wheel(0, delta)

    elif action_name == "key_combination":
        page.keyboard.press("+".join(action_args["keys"]))

    elif action_name == "hover_at":
        x, y = scale_coordinate(action_args["coordinate"], SCREEN_WIDTH, SCREEN_HEIGHT)
        page.mouse.move(x, y)

    elif action_name == "drag_and_drop":
        sx, sy = scale_coordinate(action_args["startCoordinate"], SCREEN_WIDTH, SCREEN_HEIGHT)
        ex, ey = scale_coordinate(action_args["endCoordinate"], SCREEN_WIDTH, SCREEN_HEIGHT)
        page.mouse.move(sx, sy)
        page.mouse.down()
        page.mouse.move(ex, ey)
        page.mouse.up()

    elif action_name == "wait":
        time.sleep(action_args.get("duration_ms", 1000) / 1000)

    return False


def run_browser_task(task: str, start_url: str, max_steps: int = 10):
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page(viewport={"width": SCREEN_WIDTH, "height": SCREEN_HEIGHT})
        page.goto(start_url)

        history = []

        for step in range(max_steps):
            screenshot = take_screenshot(page)

            # Build contents for this step
            user_content = {
                "role": "user",
                "parts": [
                    Part.from_text(task if step == 0 else "What should I do next?"),
                    Part.from_bytes(data=screenshot, mime_type="image/png"),
                ],
            }

            response = client.models.generate_content(
                model="gemini-3-flash",
                contents=history + [user_content],
                config=GenerateContentConfig(
                    tools=[Tool(computer_use=ComputerUse(environment=Environment.ENVIRONMENT_BROWSER))],
                    safety_acknowledgement=True,
                ),
            )

            # Update history
            history.append(user_content)
            history.append({"role": "model", "parts": response.candidates[0].content.parts})

            # Check if done
            if response.candidates[0].finish_reason.name == "STOP":
                print(f"Task complete: {response.text}")
                break

            # Execute all function calls
            for part in response.candidates[0].content.parts:
                if part.function_call:
                    action_name = part.function_call.name
                    action_args = dict(part.function_call.args)
                    print(f"  Executing: {action_name}({action_args})")

                    # ⚠️ In production: show action to user and get confirmation before executing
                    execute_action(page, action_name, action_args)
                    time.sleep(0.5)  # wait for page to settle

        browser.close()


def scale_coordinate(coord: list, screen_width: int, screen_height: int) -> tuple:
    x = int(coord[0] * screen_width / 1000)
    y = int(coord[1] * screen_height / 1000)
    return x, y


# Run the task
run_browser_task(
    task="Go to google.com and search for 'Vertex AI'",
    start_url="about:blank",
)
```

---

## § 4. Safety Requirements

### When: deploying computer use in any environment

> **Computer Use can perform real actions on real systems. Always require human confirmation.**

```python
def confirm_action(action_name: str, action_args: dict) -> bool:
    """Show the action to the user and require explicit confirmation."""
    print(f"\n[ACTION REQUIRED] Gemini wants to: {action_name}")
    print(f"  Parameters: {action_args}")
    confirm = input("  Allow this action? (yes/no): ").strip().lower()
    return confirm == "yes"

# In the action loop:
for part in response.candidates[0].content.parts:
    if part.function_call:
        action_name = part.function_call.name
        action_args = dict(part.function_call.args)

        if confirm_action(action_name, action_args):   # always confirm
            execute_action(page, action_name, action_args)
        else:
            print("Action skipped by user.")
            break
```

### Safety Rules

| Rule | Description |
|------|-------------|
| `safety_acknowledgement=True` | Must be set on every request — no exceptions |
| Human confirmation | Show every action to user before executing in production |
| Scope limitation | Restrict browser to specific domains when possible |
| Sensitive data | Never pass passwords or API keys in task descriptions |
| Audit logging | Log all actions executed for review |

> **Rule:**
> - Computer Use is a Preview feature — not suitable for autonomous production deployments.
> - Never run Computer Use in fully automated mode without human oversight.
> - Treat every action as potentially irreversible (form submission, purchases, deletions).
