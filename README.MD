# playwright-request-mocker

*Automatically generate and use network request mocks inside Playwright!*

### Features :mag_right:

Mocking your API requests takes too much precious development time, this library strives to make it effortless by:
- Allowing you to declare just once the hook use, it finds the mock file;
- If the mock file does not exist yet, it will open a Playwright's chromium tab, and record all your XHR requests and responses;
- It'll automatically intercept all registered HTTP requests defined in the mock file for any future runs.


### Install :construction_worker:
1. Install the package:
  -   ``npm install playwright-request-mocker -D``
  -   or ``yarn add playwright-request-mocker -D``;
2. Be sure to have ``@playwright/test`` also installed.

### Using :electric_plug:
1. Add to a ``.spec`` file, inside a ``beforeEach`` or ``test`` method, the hook call ``useNetworkRecordMocks`` passing the test context page, identifier of the mock (only necessary if each test scenario has a different mock), and a route to be used by the recording tab if there is no mock file yet;

    - e.g.
```
import { useNetworkRecordMocks } from 'playwright-request-mocker';          // if using .mjs / .ts
// const { useNetworkRecordMocks } = require('playwright-request-mocker');  //if using .js

test.describe('Your test', () => {
  // If your network requests/responses are the same for every test scenario, define it here.
  test.beforeEach(async ({ page }) => {
    await useNetworkRecordMocks(page, {
      recordRoute: `${process.env.APP_URL}/page-you-are-testing`,
    });

    await page.goto(`${process.env.APP_URL}/page-you-are-testing`);
  });

  // else use it here if each test scenario expects different results.
  test('scenario1', async ({ page }) => {
    // It'll generate a new file if it does not exist (".spec.scenario1.mocks.json")
    // then it'll read it and mock all defined network requests.
    await useNetworkRecordMocks(page, {
      identifier: 'scenario1',
      recordRoute: `${process.env.APP_URL}/page-you-are-testing`
    });

    await page.goto(`${process.env.APP_URL}/page-you-are-testing`);

    //... your steps and asserts.
  });
});
```

1. Run in debug mode;
- PWDEBUG=console forces Playwright to run headed, disables the timeouts and shows the console helper. 

```
// Linux/macOS
PWDEBUG=console npm run playwright test

// Windows on cmd.exe
set PWDEBUG=console
npm run playwright test

// Windows on PowerShell
$env:PWDEBUG="console"
npm run playwright test
```

- If the mock file does not exist yet, it'll open a new tab and will be recording all the XHR requests as you navigate. When you think you recorded everything you needed, press the resume button in the ``playwright/test`` UI.
- After it, or if the mock file ever exists, it will use the results to run your test scenario.


### How it works :question:

- useNetworkRecordMocks checks if there is a file ``[invoker-file-name].mocks.json`` (or ``[invoker-file-name].[identifier].mocks.json``), if it doesn't exists, it'll open a chromiun tab using chrome's [recordHAR](https://en.wikipedia.org/wiki/HAR_(file_format)) option. When the Playwright debug mode is unpaused, it'll process such file removing every request other than XHR ones;
- useNetworkRecordMocks will parse the existing file, and mock every request declared on it.

### Different use cases :footprints:

- **For a given test scenario, I need to change the mocked value**: there are 3 ways to achieve this, 1 being creating a new specific mock file at each scenario (by calling useNetworkRecordMocks inside each test with a identifier), and others being overriding the route response, like the following:

Overriding approach: for when the ``useNetworkRecordMocks`` call is inside the test scenario:
```
  import { useNetworkRecordMocks, mockRouteResponse } from 'playwright-request-mocker';

  // [...]

  test('my different scenario', async ({
    page,
  }) => {
    const errorMessage = 'error message I expect';

    await useNetworkRecordMocks(page, {
      recordRoute: `${process.env.APP_URL}/page-you-are-testing`,
      overrideResponses: {
        "/ENDPOINT_TO_CHANGE": { errors: [{ message: errorMessage }] }
      }
    });

    await page.goto(`${process.env.APP_URL}/page-you-are-testing`);

    await Promise.all([
      page.waitForRequest('**/ENDPOINT_TO_CHANGE'),
      fillForm(page),
    ]);

    await page.waitForSelector(`text=${errorMessage}`);
    const visible = await page.isVisible(`text=${errorMessage}`);
    expect(visible).toBeTruthy();
  });
```
   Unrouting approach: for when the ``useNetworkRecordMocks`` call is inside a ``beforeEach``:
```

test.describe('Test', () => {
  test.beforeEach(async ({ page }) => {
    await useNetworkRecordMocks(page, {
      recordRoute: `${process.env.APP_URL}/page-you-are-testing`,
    });

    await page.goto(`${process.env.APP_URL}/page-you-are-testing`);
  });

  test('my different scenario', async ({
    page
  }) => {
    const errorMessage = 'error message I expect';

    page.unroute('**/ENDPOINT_TO_CHANGE');
    mockRouteResponse(
      page,
      '**/ENDPOINT_TO_CHANGE',
      { errors: [{ message: errorMessage }] },
      400,
    );

    await Promise.all([
      page.waitForRequest('**/ENDPOINT_TO_CHANGE'),
      fillForm(page),
    ]);

    await page.waitForSelector(`text=${errorMessage}`);
    const visible = await page.isVisible(`text=${errorMessage}`);
    expect(visible).toBeTruthy();
  });
}
```

- **I want to assert the payload sent against what was expected**: ``useNetworkRecordMocks`` returns every mocked request payload, you can use it to assert the values:

```
  test('my different scenario', async ({
    page,
  }) => {

    const requests = await useNetworkRecordMocks(
      page,
      {
        recordRoute: `${process.env.APP_URL}/test-route`,
      }
    );

    const [req] = await Promise.all([
      page.waitForRequest('**/ENDPOINT_TO_TEST'),
      fillForm(page),
    ]);

    const expectedPayload = requests.find(r => r.url.includes('ENDPOINT_TO_TEST'))?.requestData;

    expect(expectedPayload).toStrictEqual(req.postDataJSON());

  });

```

- **I already have a HAR recorded file that I want to use**: as long as it follows the same name structure as the file that is using it, the lib will prioritize it over recording a new one to generate the new mocks.json file:
```
  /tests
  |- my-test.spec.js
  |- my-test.spec.har
  |- my-test.spec.identifier.har

```

----


Feel free to contribute, report bugs, or [contact me](https://github.com/kousenlsn).
