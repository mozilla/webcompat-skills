## Interventions Tests

### File Structure

All tests are located in `testing/webcompat/interventions/tests/` with the following naming convention:

```
testing/webcompat/interventions/tests/test_{bug_id}_{domain_underscores}.py
```

### Structure
Each test file has two test cases:
- **test_disabled** — intervention OFF, confirms the bug still exists
  (e.g. a block message appears, layout is broken)
- **test_enabled** — intervention ON, confirms the fix works
  (e.g. block message gone, page loads normally)

### What's possible to test
Most tests load a URL and check if a specific element or text
is present/absent. Tests can also:
- Check HTTP requests/responses
- Verify event listeners are added or fired
- Test redirects, scrolling, keyboard input
- Handle captchas/logins semi-automatically (automate STR, let user
  handle the captcha/login manually)
- Compare before/after screenshots of an element
- Use chrome JS and WebDriver APIs for advanced checks

### Examples
Look at existing tests in `testing/webcompat/interventions/tests/` for patterns.
When creating a new test, find a similar existing test and use it as a template.

Here are a few examples:

1. An archetypical simple test for a user-agent override for a browser block (with a platform requirement):
```
import pytest
URL = "https://example.com/"
SUPPORTED_CSS = "#screen1"
UNSUPPORTED_TEXT = "WE ARE CURRENTLY NOT SUPPORTING YOUR BROWSER"


@pytest.mark.skip_platforms("android")
@pytest.mark.asyncio
@pytest.mark.with_interventions
async def test_enabled(client):
    await client.navigate(URL, wait="none")
    assert client.await_css(SUPPORTED_CSS, is_displayed=True)
    assert not client.find_text(UNSUPPORTED_TEXT, is_displayed=True)


@pytest.mark.skip_platforms("android")
@pytest.mark.asyncio
@pytest.mark.without_interventions
async def test_disabled(client):
    await client.navigate(URL, wait="none")
    assert client.await_text(UNSUPPORTED_TEXT, is_displayed=True)
    assert not client.find_css(SUPPORTED_CSS, is_displayed=True)
```

2. Testing scrolling:


```
import asyncio
import pytest
URL = "https://example.com/"

async def check_if_scrolling_works(client):
    await client.navigate(URL)
    body = client.await_css("body")
    expected_pos = client.execute_script("return window.scrollY")
    client.apz_scroll(body, dy=100000)
    await asyncio.sleep(0.2)
    final_pos = client.execute_script("return window.scrollY")
    return expected_pos != final_pos


@pytest.mark.only_platforms("android")
@pytest.mark.asyncio
@pytest.mark.with_interventions
async def test_enabled(client, in_headless_mode):
    assert await check_if_scrolling_works(client)


@pytest.mark.only_platforms("android")
@pytest.mark.asyncio
@pytest.mark.without_interventions
async def test_disabled(client, in_headless_mode):
    assert not await check_if_scrolling_works(client)
```
3. Detecting a page reload cycle:

```
import asyncio
import pytest

URL = "https://example.com/"

async def is_reload_cycle_detected(client):
    await client.navigate(URL)
    client.await_css("#alwaysFetch")
    client.execute_script("location.reload()")
    await asyncio.sleep(1)
    try:
        await (await client.promise_navigation_begins(URL, timeout=3))
        return True
    except asyncio.exceptions.TimeoutError:
        return False


@pytest.mark.only_platforms("android")
@pytest.mark.asyncio
@pytest.mark.with_interventions
async def test_enabled(client):
    assert not await is_reload_cycle_detected(client)


@pytest.mark.only_platforms("android")
@pytest.mark.asyncio
@pytest.mark.without_interventions
async def test_disabled(client):
    assert await is_reload_cycle_detected(client)

```

### Running Tests

```
./mach test-interventions --headless --bug {bug_id}
```

- `--bug/-b` matches strings in filenames, not just bug numbers
- To run multiple tests: `./mach test-interventions -b nintendo axisbank 1819450`
- Omit `--headless` if tests hang (easier to close the browser window)
- Screenshots are taken on failure by default (disable with `--no-failure-screenshots/-s`)

### Android Notes
- Android interventions use GeckoView demo browser by default (no full Fenix build needed)
- Skip APK rebuild when only changing tests: `MOZ_DISABLE_ADB_INSTALL=True ./mach test-interventions ...`
- Most Android interventions can be developed/tested in responsive design mode

### Annotations
- `@pytest.mark.with_interventions` / `@pytest.mark.without_interventions`
- `@pytest.mark.only_platforms("android")` / `@pytest.mark.skip_platforms("android")`
- Do NOT add platform markers unless the user explicitly requests them

### Limitations — When Tests May Not Be Possible
- Site detects and blocks WebDriver
- Issue is too intermittent
- Login/captcha/2FA/VPN requirements (though semi-automated tests can handle these)
